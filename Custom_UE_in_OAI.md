# OAI — Adding a Custom UE: Runbook & Troubleshooting Log

A precise record of adding a second subscriber (`208990100001189`) to the running
5G core and getting it to register. Two parts: **(1) the commands to run, in order**,
and **(2) every problem we hit, its cause, and the fix/check.**

- Deployment dir: `~/OAI_Begineers_Experiment/openairinterface5g/ci-scripts/yaml_files/5g_rfsimulator/`
- New UE IMSI: `208990100001189` (PLMN 208/99), same Key/OPc as the test subscribers
- DB creds (from AMF config): user `test` / pass `test`, root pass `linux`, database `oai_db`
- Subscriber table in this deployment: **`users`** (legacy schema)

---

## PART 1 — Commands in order to run

### A. One-time setup of the new UE

```bash
# 1. Create the UE SIM config (copy an existing one, change IMSI)
cd ~/OAI_Begineers_Experiment/openairinterface5g/ci-scripts/conf_files
cp nrue.uicc.conf nrue.uicc2.conf
#   edit nrue.uicc2.conf -> imsi = "208990100001189"  (key/opc/dnn unchanged)

# 2. Add a matching subscriber row to the seed file (see Problem 5 for correct placement)
cd ../yaml_files/5g_rfsimulator
#   in oai_db.sql, right AFTER the last pre-seeded users row (…115), add:
#   INSERT INTO `users` VALUES ('208990100001189','1','55000000000000',NULL,'PURGED',50,40000000,100000000,47,0000000000,1,0xfec86ba6eb707ed08905757b1bb44b8f,0,0,0x40,'ebd07771ace8677a',0xc42449363bbad02b66d16bc975d77cc1);

# 3. Add a new UE service (oai-nr-uenew) in docker-compose.yaml,
#    copying the oai-nr-ue block; point it at nrue.uicc2.conf and set --uicc0.imsi 208990100001189
#    (see Problem 1 for the option that must NOT be truncated)
```

### B. Bring the system up — ALWAYS in this order

```bash
cd ~/OAI_Begineers_Experiment/openairinterface5g/ci-scripts/yaml_files/5g_rfsimulator

# 1. CORE FIRST
docker compose up -d mysql oai-amf oai-smf oai-upf oai-ext-dn

# 2. WAIT until BOTH mysql and oai-amf are (healthy) — do NOT use a fixed sleep.
#    Either re-run this until healthy:
docker compose ps
#    ...or block automatically:
until [ "$(docker inspect -f '{{.State.Health.Status}}' rfsim5g-mysql)" = healthy ] \
   && [ "$(docker inspect -f '{{.State.Health.Status}}' rfsim5g-oai-amf)" = healthy ]; do
  echo "waiting for mysql + amf..."; sleep 5
done

# 3. gNB, confirm it connects to the AMF
docker compose up -d oai-gnb && sleep 10
docker logs rfsim5g-oai-amf 2>&1 | grep -A6 "gNBs"        # expect "Connected"

# 4. UE last
docker compose up -d oai-nr-uenew && sleep 20
docker logs rfsim5g-oai-amf 2>&1 | grep -iE "208990100001189|authentic|reject" | tail -30
```

### C. Verify the UE works

```bash
# registered?  -> expect 5GMM-REGISTERED for 208990100001189
docker logs rfsim5g-oai-amf 2>&1 | grep "208990100001189" | tail -5

# got an IP?  -> expect inet 12.1.1.x
docker exec -it rfsim5g-oai-nr-uenew ip addr show oaitun_ue1

# data path works?  -> expect 0% packet loss
EXTDN=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}' rfsim5g-oai-ext-dn | tr ' ' '\n' | grep '192.168.72')
docker exec -it rfsim5g-oai-nr-uenew ping -I oaitun_ue1 -c 5 "$EXTDN"
```

### D. Verify / fix the DB row when needed

```bash
# check the row exists with correct key/opc
docker exec -i rfsim5g-mysql mysql -u test -ptest oai_db \
  -e "SELECT imsi, HEX(\`key\`), HEX(opc) FROM users WHERE imsi='208990100001189';"

# reliable way to insert it live: clone a known-good row and change only the IMSI
docker exec -i rfsim5g-mysql mysql -u test -ptest oai_db <<'SQL'
DELETE FROM users WHERE imsi='208990100001189';
CREATE TEMPORARY TABLE tmp_u AS SELECT * FROM users WHERE imsi='208990100001100';
UPDATE tmp_u SET imsi='208990100001189';
INSERT INTO users SELECT * FROM tmp_u;
SQL
```

---

## PART 2 — Problems we hit (cause → fix / check)

### Problem 1 — UE container exits immediately with code 1

**Symptom:** log shows `[CONFIG] unknown option: --log_config.global_log_opt>` then
`Main child exited normally (with status '1')`. No registration attempt at all.

**Cause:** when copying the `USE_ADDITIONAL_OPTIONS` line into the new UE service in
`docker-compose.yaml`, the long line got **truncated** (ended in `...global_log_opt>`
with a stray `>`). The UE rejects the unknown option at startup, before touching the radio.

**Fix:** make the new service's `USE_ADDITIONAL_OPTIONS` identical to the working
`oai-nr-ue` service, changing only the IMSI. The option must end fully, e.g.
`... --log_config.global_log_options level nocolor time`. Or just delete the broken
fragment (it's cosmetic). Then:
```bash
docker compose up -d --force-recreate oai-nr-uenew
docker compose logs -f oai-nr-uenew
```

### Problem 2 — Registration Reject with NO authentication step

**Symptom:** AMF goes `5GMM-DEREGISTERED → COMM-PROC-INIT → REGISTRATION_REJECT_SENT`
in ~2 ms; no "Authentication Request/Response" lines in between.

**Cause:** the AMF could not obtain authentication vectors — it never even
challenged the UE. This is a **core/DB problem, not a UE problem.** (A correct run
shows Security Vectors → Auth Request → Auth Response → Security Mode → Accept.)

**Check:** look for the real reason right before the reject:
```bash
docker logs rfsim5g-oai-amf 2>&1 | grep -iE "authentic|vector|mysql|db|reject" | tail -20
```

### Problem 3 — `Can't connect to MySQL server on 'mysql:3306' (111)` (THE main issue)

**Symptom:** `[amf_authentication] [error] ... (111)` → `Cannot connect to MySQL DB`
→ `Request Authentication Vectors failure` → reject.

**Cause:** the AMF asked MySQL for vectors **before MySQL was ready to accept
connections.** Happened every time we started the UE too soon — worst right after
`docker compose down -v`, because a fresh volume takes 60–90 s to initialize.
`docker compose restart` (all containers at once) causes the same race.

**Fix:** never rely on a fixed `sleep`; **wait for `mysql` AND `oai-amf` to be
`(healthy)`** before starting the gNB/UE. Use Part 1-B step 2. Order is always:
**core → wait healthy → gNB → UE.** Avoid `docker compose restart <whole stack>`.

### Problem 4 — Reject even though the DB row looks correct

**Symptom:** `Receive information from MySQL with IMSI 208990100001189` appears
(so the row was read), but the UE still ends `5GMM-DEREGISTERED`.

**Cause / how we isolated it:** to remove all doubt about the subscriber data, we
**cloned a known-good row** (`…100`, which registers) and changed only the IMSI, so
Key/OPc/SQN were byte-for-byte identical to a working UE. It *still* failed — which
proved the subscriber data was never the problem and pointed back to the MySQL
timing race (Problem 3). Verify the clone:
```bash
docker exec -i rfsim5g-mysql mysql -u test -ptest oai_db \
  -e "SELECT imsi, HEX(\`key\`), HEX(opc) FROM users WHERE imsi IN ('208990100001100','208990100001189');"
# both rows must show identical key FEC86…B8F and opc C42449…7CC1
```

### Problem 5 — Edited `oai_db.sql` but the change doesn't take effect

**Symptom:** added the INSERT to `oai_db.sql`, but the running DB doesn't reflect it
(or the row sits in a fragile spot near `ALTER TABLE ... ENABLE KEYS`).

**Cause:** two things — (a) editing `oai_db.sql` only re-seeds MySQL if the volume is
wiped (`docker compose down -v`); a plain restart keeps the old data. (b) the new line
was placed at the end of the dump block instead of within the `users` insert section.

**Fix:** place the new row right after the last pre-seeded `users` row and remove the
stray `;` after the comment. Find the anchor line:
```bash
grep -n "208990100001115" oai_db.sql     # e.g. line 209 — put your INSERT right after it
```
For a change to load, do a full reset: `docker compose down -v` then bring up staged
(Part 1-B). For a quick live add without reset, use the clone command (Part 1-D).

### Problem 6 — `-bash: PW: No such file or directory` when checking the DB

**Cause:** the command literally contained `-p<PW>`; the shell tried to run `PW`.

**Fix:** substitute the real password: `-ptest` (user `test`) — no space after `-p`:
```bash
docker exec -i rfsim5g-mysql mysql -u test -ptest oai_db -e "SELECT imsi FROM users WHERE imsi='208990100001189';"
```

---

## The one lesson to remember

**Almost every failure here was one root cause: the AMF needed MySQL before MySQL
was ready.** The UE config and the subscriber row were correct the whole time.
The reliable routine, every session:

```
core up  →  WAIT for mysql + amf (healthy)  →  gNB up  →  UE up  →  verify
```

Success looks like: AMF shows `5GMM-REGISTERED` for the IMSI, the UE has
`oaitun_ue1` with a `12.1.1.x` IP, and ping through it is `0%` loss.
```
