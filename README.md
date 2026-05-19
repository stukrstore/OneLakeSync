# On-Prem Hadoop → Azure (OneLake / ADLS Gen2) 파일 동기화 가이드

> 최신 검증 버전: **Hadoop 3.4.1** + **hadoop-azure 3.4.1** + **azure-storage 8.6.6**
> 대상 스토리지: **OneLake (Microsoft Fabric)** 또는 **ADLS Gen2 (HNS enabled)**
> 인증: **Entra ID Service Principal (OAuth 2.0 Client Credentials)**

---

## 1. Entra ID 앱 등록 및 자격증명 확인

Hadoop ABFS 드라이버가 사용할 **Service Principal(앱 등록)** 을 만들고, `client id` / `tenant id` / `client secret` 세 값을 확보합니다. Portal / Azure CLI 둘 다 가능합니다.

### 1-1. Azure Portal 로 등록

1. **Microsoft Entra ID → App registrations → New registration**
   - Name: `sp-hadoop-onelake-sync` (자유)
   - Supported account types: **Accounts in this organizational directory only** (단일 테넌트 권장)
   - Redirect URI: 비워둠 (Daemon/Service 용도)
2. 등록 직후 **Overview** 페이지에서 아래 값을 복사
   - **Application (client) ID** → `<APP_CLIENT_ID>` 로 사용
   - **Directory (tenant) ID** → `<TENANT_ID>` 로 사용
   - **Object ID** → RBAC 부여 시 `--assignee-object-id` 로 사용 (선택)
3. **Certificates & secrets → Client secrets → + New client secret**
   - Description: `hadoop-abfs`
   - Expires: 조직 정책에 맞게 (예: 180 days / 365 days, 최대 24개월)
   - **Value** 컬럼의 문자열을 즉시 복사 → `<APP_CLIENT_SECRET>` (페이지 이탈 후에는 다시 볼 수 없음)
4. 만료일을 캘린더에 등록하고, 만료 30일 전 회전(rotate) 절차를 마련

> Secret ID 는 노출되어도 무방하지만 **Value** 는 비밀번호 수준으로 다뤄야 합니다. 가능하면 본 가이드 **4-3** 의 `hadoop credential` (JCEKS) 로 보관하세요.

### 1-2. Azure CLI 로 등록 (자동화 권장)

```bash
# 0) 로그인 / 구독 선택
az login
az account set --subscription "<SUBSCRIPTION_ID_OR_NAME>"

# 1) Tenant ID 확인
TENANT_ID=$(az account show --query tenantId -o tsv)
echo "TENANT_ID=$TENANT_ID"

# 2) 앱 등록 + Service Principal 동시 생성
APP_NAME="sp-hadoop-onelake-sync"
APP_JSON=$(az ad sp create-for-rbac \
  --name "$APP_NAME" \
  --skip-assignment \
  --years 1 \
  -o json)

CLIENT_ID=$(echo "$APP_JSON"   | jq -r .appId)
CLIENT_SECRET=$(echo "$APP_JSON" | jq -r .password)
SP_OBJECT_ID=$(az ad sp show --id "$CLIENT_ID" --query id -o tsv)

cat <<EOF
TENANT_ID      = $TENANT_ID
CLIENT_ID      = $CLIENT_ID
CLIENT_SECRET  = $CLIENT_SECRET   # 1회성 — 안전한 곳에 즉시 보관
SP_OBJECT_ID   = $SP_OBJECT_ID
EOF
```

> `--skip-assignment` 로 일단 만들고, 권한은 ADLS Gen2 / Fabric workspace 단위로 따로 부여합니다 (본 가이드 **7-3** 참고).

### 1-3. 기존 앱에서 값만 다시 확인하기

이미 만들어 둔 앱의 `client id` / `tenant id` 를 다시 확인:

```bash
# Display Name 으로 검색
az ad app list --display-name "sp-hadoop-onelake-sync" \
  --query "[].{appId:appId, displayName:displayName, tenantId:'$(az account show --query tenantId -o tsv)'}" -o table

# 또는 Portal: Entra ID → App registrations → 앱 클릭 → Overview
```

> **Client Secret 의 Value 는 생성 시점에만 노출**되므로, 분실했다면 새 secret 을 발급(`Certificates & secrets → New client secret` 또는 `az ad app credential reset --id <CLIENT_ID> --years 1`)받아야 합니다.

```bash
# CLI 로 secret 재발급 (기존 secret 모두 무효화하려면 --append 제거)
az ad app credential reset \
  --id "$CLIENT_ID" \
  --append \
  --years 1 \
  --display-name "hadoop-abfs-rotate-$(date +%F)"
# 출력의 password 값이 새 client secret
```

### 1-4. 빠른 자격증명 검증 (Hadoop 설정 전)

`curl` 로 토큰이 실제 발급되는지 미리 확인하면 이후 ABFS 401 원인 분리에 도움이 됩니다.

```bash
curl -sS -X POST \
  "https://login.microsoftonline.com/${TENANT_ID}/oauth2/v2.0/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=${CLIENT_ID}" \
  -d "client_secret=${CLIENT_SECRET}" \
  -d "scope=https://storage.azure.com/.default" \
  | jq '{token_type, expires_in, access_token: (.access_token[0:40] + "...")}'
```

| scope | 용도 |
| --- | --- |
| `https://storage.azure.com/.default` | ADLS Gen2 (`*.dfs.core.windows.net`) |
| `https://storage.azure.com/.default` | OneLake 도 동일 — OneLake 는 storage.azure.com audience 토큰 수용 |

> 200 OK 와 `access_token` 이 보이면 SP 자체는 정상. 이후 401/403 은 **권한(RBAC/ACL)** 또는 **core-site.xml 오타** 문제로 좁혀집니다.

---

## 2. Hadoop Optional Tools 등록

`hadoop-azure`, `hadoop-azure-datalake` 모듈을 classpath 에 추가합니다.

- 파일 위치: `/usr/local/hadoop/etc/hadoop/hadoop-env.sh`

```bash
export HADOOP_OPTIONAL_TOOLS="hadoop-azure,hadoop-azure-datalake"
```

> 위 한 줄만 등록해도 `share/hadoop/tools/lib/*` 의 jar 가 자동으로 로딩됩니다. 별도 jar 복사는 버전 충돌(특히 `jetty-util*`, `wildfly-openssl`) 을 유발할 수 있으므로 가능한 피하십시오.

---

## 3. 필수 라이브러리 (수동 설치가 필요한 경우)

배포판이 `tools/lib` 를 포함하지 않는 경우에만 아래 jar 를 받아 `/usr/local/hadoop/share/hadoop/tools/lib/` (없으면 `/usr/local/hadoop/lib/`) 에 배치합니다.

```bash
HADOOP_VERSION=3.4.1
LIB_DIR=/usr/local/hadoop/share/hadoop/tools/lib

cd "$LIB_DIR"

# Hadoop ABFS / ADLS 드라이버
wget https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-azure/${HADOOP_VERSION}/hadoop-azure-${HADOOP_VERSION}.jar
wget https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-azure-datalake/${HADOOP_VERSION}/hadoop-azure-datalake-${HADOOP_VERSION}.jar

# Azure Storage SDK (legacy v8 — hadoop-azure 가 사용)
wget https://repo1.maven.org/maven2/com/microsoft/azure/azure-storage/8.6.6/azure-storage-8.6.6.jar
wget https://repo1.maven.org/maven2/com/microsoft/azure/azure-data-lake-store-sdk/2.3.10/azure-data-lake-store-sdk-2.3.10.jar

# Keyvault / OAuth 의존성
wget https://repo1.maven.org/maven2/com/microsoft/azure/azure-keyvault-core/1.2.6/azure-keyvault-core-1.2.6.jar

# ABFS SSL 가속 (선택)
wget https://repo1.maven.org/maven2/org/wildfly/openssl/wildfly-openssl/2.2.5.Final/wildfly-openssl-2.2.5.Final.jar

# JSON / HTTP 의존성 (기존 Hadoop 에 없을 때만)
wget https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-util-ajax/9.4.53.v20231009/jetty-util-ajax-9.4.53.v20231009.jar
```

설치 후 확인:

```bash
hadoop classpath | tr ':' '\n' | grep -Ei 'azure|datalake'
```

---

## 4. OAuth 2.0 Client Credentials 설정

- 파일 위치: `/usr/local/hadoop/etc/hadoop/core-site.xml`
- 참조: https://hadoop.apache.org/docs/r3.4.1/hadoop-azure/abfs.html#OAuth_2.0_Client_Credentials

### 4-1. 단일 계정 전용 설정

```xml
<configuration>
  <property>
    <name>fs.azure.account.auth.type</name>
    <value>OAuth</value>
  </property>
  <property>
    <name>fs.azure.account.oauth.provider.type</name>
    <value>org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider</value>
  </property>
  <property>
    <name>fs.azure.account.oauth2.client.endpoint</name>
    <value>https://login.microsoftonline.com/<TENANT_ID>/oauth2/token</value>
  </property>
  <property>
    <name>fs.azure.account.oauth2.client.id</name>
    <value><APP_CLIENT_ID></value>
  </property>
  <property>
    <name>fs.azure.account.oauth2.client.secret</name>
    <value><APP_CLIENT_SECRET></value>
  </property>

  <!-- ABFS 권장 옵션 -->
  <property>
    <name>fs.azure.createRemoteFileSystemDuringInitialization</name>
    <value>false</value>
    <description>OneLake/Fabric 사용 시에는 반드시 false (파일시스템=Lakehouse 가 미리 생성되어 있음)</description>
  </property>
  <property>
    <name>fs.azure.enable.check.access</name>
    <value>true</value>
  </property>
  <property>
    <name>fs.azure.ssl.channel.mode</name>
    <value>OpenSSL</value>
  </property>
</configuration>
```

### 4-2. 계정별 분기 설정 (여러 스토리지 / Fabric workspace 사용 시)

`<account>.dfs.core.windows.net` 또는 `<workspace>@onelake.dfs.fabric.microsoft.com` 단위로 키를 분리할 수 있습니다.

```xml
<property>
  <name>fs.azure.account.auth.type.onelake.dfs.fabric.microsoft.com</name>
  <value>OAuth</value>
</property>
<property>
  <name>fs.azure.account.oauth.provider.type.onelake.dfs.fabric.microsoft.com</name>
  <value>org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider</value>
</property>
<property>
  <name>fs.azure.account.oauth2.client.endpoint.onelake.dfs.fabric.microsoft.com</name>
  <value>https://login.microsoftonline.com/<TENANT_ID>/oauth2/token</value>
</property>
<property>
  <name>fs.azure.account.oauth2.client.id.onelake.dfs.fabric.microsoft.com</name>
  <value><APP_CLIENT_ID></value>
</property>
<property>
  <name>fs.azure.account.oauth2.client.secret.onelake.dfs.fabric.microsoft.com</name>
  <value><APP_CLIENT_SECRET></value>
</property>
```

### 4-3. Secret 을 평문으로 두지 않기 — Hadoop Credential Provider

```bash
# JCEKS 파일에 시크릿 저장
hadoop credential create fs.azure.account.oauth2.client.secret \
  -value '<APP_CLIENT_SECRET>' \
  -provider jceks://file/etc/hadoop/conf/azure.jceks

chmod 640 /etc/hadoop/conf/azure.jceks
```

`core-site.xml` 에 provider 위치만 등록하고 평문 secret 은 제거:

```xml
<property>
  <name>hadoop.security.credential.provider.path</name>
  <value>jceks://file/etc/hadoop/conf/azure.jceks</value>
</property>
```

---

## 5. URI 규칙

| 대상 | URI 형식 |
| --- | --- |
| ADLS Gen2 | `abfss://<container>@<account>.dfs.core.windows.net/<path>` |
| OneLake (Fabric) | `abfss://<workspace>@onelake.dfs.fabric.microsoft.com/<lakehouse>.Lakehouse/<Files\|Tables>/<path>` |

> OneLake 의 workspace 이름에 공백 / 특수문자가 있으면 **GUID** 또는 **`__` (이중 언더스코어 prefix)** 형태의 정규화된 이름을 사용하세요. (예: `__WS_FabricWorkshop`)

---

## 6. Sample Hadoop Commands

### 6-1. 디렉터리 조회

```bash
# OneLake Files 영역 조회
hadoop fs -ls abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/Upload/

# 재귀 조회 + 사람이 읽기 좋은 사이즈
hadoop fs -ls -R -h abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/

# 용량 합계
hadoop fs -du -s -h abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/Upload/
```

### 6-2. 디렉터리 생성 / 삭제

```bash
hadoop fs -mkdir -p abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/Upload/2026/05/

# skipTrash 로 즉시 삭제 (OneLake 는 trash 미지원)
hadoop fs -rm -r -skipTrash abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/Upload/old/
```

### 6-3. 단일 파일 업로드 / 다운로드

```bash
# put: 로컬 → OneLake
hadoop fs -put -f ./enu_sql_server_2022_developer_edition_x64_dvd_7cacf733.iso \
  abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/Upload/

# copyFromLocal / copyToLocal
hadoop fs -copyFromLocal -f ./data/*.parquet \
  abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/raw/

hadoop fs -copyToLocal \
  abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/Upload/report.csv \
  ./report.csv

# checksum 비교 (무결성 확인)
hadoop fs -checksum abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/Upload/report.csv
md5sum ./report.csv
```

### 6-4. DistCp — 대용량 / 디렉터리 전체 동기화

기본 사용:

```bash
hadoop distcp \
  -m 20 -bandwidth 50 \
  -update -delete \
  hdfs:///user/etl/landing/2026-05-19/ \
  abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/landing/2026-05-19/
```

옵션 의미:

| Flag | 설명 |
| --- | --- |
| `-m 20` | 최대 mapper 수 (병렬 전송) |
| `-bandwidth 50` | mapper 당 MB/s 상한 |
| `-update` | 크기/타임스탬프 다르면 덮어쓰기 (증분 동기화) |
| `-delete` | source 에 없는 target 파일 제거 (미러) |
| `-pugp` | permission / user / group / preserve 보존 |
| `-strategy dynamic` | 큰 파일 위주 분할 — 매우 큰 파일에 권장 |
| `-numListstatusThreads 40` | 디렉터리 listing 가속 |
| `-i` | I/O 오류 무시하고 계속 |
| `-log <path>` | distcp 자체 로그 위치 |

### 6-5. DistCp — 로컬 → OneLake 직접 전송

`hdfs://` 가 아닌 로컬 파일도 `file://` 스킴으로 distcp 사용 가능합니다.

```bash
hadoop distcp \
  -m 8 \
  file:///data/iso/enu_sql_server_2022_developer_edition_x64_dvd_7cacf733.iso \
  abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/Upload/
```

### 6-6. DistCp — 명령행에서 OAuth 자격증명 주입 (config 파일 없이)

```bash
hadoop distcp \
  -D fs.azure.account.auth.type=OAuth \
  -D fs.azure.account.oauth.provider.type=org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider \
  -D fs.azure.account.oauth2.client.endpoint=https://login.microsoftonline.com/<TENANT_ID>/oauth2/token \
  -D fs.azure.account.oauth2.client.id=<APP_CLIENT_ID> \
  -D fs.azure.account.oauth2.client.secret=<APP_CLIENT_SECRET> \
  -D fs.azure.createRemoteFileSystemDuringInitialization=false \
  -update -m 16 -bandwidth 80 -strategy dynamic \
  file:///data/iso/ \
  abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/Upload/
```

### 6-7. DistCp — 클라우드 ↔ 클라우드 (ADLS Gen2 → OneLake)

```bash
hadoop distcp \
  -update -m 32 -bandwidth 100 -strategy dynamic \
  -numListstatusThreads 40 \
  abfss://raw@stadlsgen2prod.dfs.core.windows.net/curated/sales/ \
  abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/sales/
```

> 같은 SP 가 두 계정 모두에 접근 가능하면 위 한 줄이면 충분합니다. 계정별로 다른 SP 가 필요하면 **4-2** 의 account-scoped 설정을 사용하세요.

### 6-8. DistCp — Snapshot / 증분 전송 (HDFS 소스)

```bash
# 1) 스냅샷 디렉토리 활성화 (HDFS admin)
hdfs dfsadmin -allowSnapshot /user/etl/landing
hdfs dfs -createSnapshot /user/etl/landing snap-2026-05-19

# 2) 최초 풀 카피
hadoop distcp -update \
  hdfs:///user/etl/landing/.snapshot/snap-2026-05-19/ \
  abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/landing/

# 3) 다음 날 — diff 만 전송
hdfs dfs -createSnapshot /user/etl/landing snap-2026-05-20
hadoop distcp \
  -diff snap-2026-05-19 snap-2026-05-20 \
  -update \
  hdfs:///user/etl/landing \
  abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/landing
```

---

## 7. 동작 확인 / 트러블슈팅

### 7-1. 토큰 발급 검증

```bash
hadoop fs -ls abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/ 2>&1 | tee /tmp/abfs.log
```

DEBUG 로그:

```bash
export HADOOP_ROOT_LOGGER=DEBUG,console
export HADOOP_OPTS="$HADOOP_OPTS -Dorg.apache.hadoop.fs.azurebfs=DEBUG"
hadoop fs -ls abfss://...
```

### 7-2. 자주 보는 오류

| 메시지 | 원인 | 조치 |
| --- | --- | --- |
| `AADToken: HTTP connection failed for getting token ... 401 Unauthorized` | client id / secret / tenant 오타, 또는 SP 가 만료된 secret 사용 | Entra ID 포털에서 secret 재발급 후 jceks 갱신 |
| `403 AuthorizationPermissionMismatch` | SP 에 RBAC/ACL 미부여 | Fabric workspace 에 **Contributor** 부여 또는 ADLS 에 **Storage Blob Data Contributor** 부여 |
| `FileSystem ... not found` (OneLake) | `fs.azure.createRemoteFileSystemDuringInitialization=true` 가 OneLake 에서 동작하지 않음 | Lakehouse 를 Fabric UI 에서 먼저 생성, 옵션은 `false` 로 |
| `No FileSystem for scheme: abfss` | hadoop-azure jar 미로딩 | `hadoop classpath` 확인, `HADOOP_OPTIONAL_TOOLS` 등록 여부 확인 |
| `SSLHandshakeException` | 사내 프록시/인증서 | `-Dhttps.proxyHost`/`-Dhttps.proxyPort` 또는 `fs.azure.ssl.channel.mode=Default` 로 변경 |

### 7-3. 권한 부여 예시

```bash
# OneLake (Fabric) — workspace 단위 (Fabric REST 또는 UI 로 부여)
# Role: Contributor 이상 (또는 Lakehouse 항목 단위 Read/Write)

# ADLS Gen2 — Azure CLI 로 RBAC
az role assignment create \
  --assignee <APP_OBJECT_ID> \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/<SUB_ID>/resourceGroups/<RG>/providers/Microsoft.Storage/storageAccounts/<ACCT>
```

---

## 8. 운영 체크리스트

- [ ] `HADOOP_OPTIONAL_TOOLS` 에 `hadoop-azure,hadoop-azure-datalake` 포함
- [ ] `core-site.xml` 에 OAuth 4종 (`auth.type`, `provider.type`, `client.endpoint`, `client.id`) + secret(jceks) 설정
- [ ] OneLake 대상이면 `fs.azure.createRemoteFileSystemDuringInitialization=false`
- [ ] SP 에 Fabric workspace / ADLS RBAC 부여 완료
- [ ] DistCp 야간 잡: `-update -delete -strategy dynamic -bandwidth <상한>` 적용
- [ ] 시크릿은 `hadoop credential` (jceks) 로만 저장, `core-site.xml` 평문 금지
- [ ] 로그 보관: `-log abfss://.../distcp-logs/$(date +%F)/`

---

## 참고 링크

- ABFS Driver (3.4.1): https://hadoop.apache.org/docs/r3.4.1/hadoop-azure/abfs.html
- DistCp Guide: https://hadoop.apache.org/docs/r3.4.1/hadoop-distcp/DistCp.html
- OneLake ABFS 경로: https://learn.microsoft.com/fabric/onelake/onelake-access-api
- Fabric workspace RBAC: https://learn.microsoft.com/fabric/get-started/roles-workspaces
