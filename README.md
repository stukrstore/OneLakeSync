# On-Prem Hadoop → Azure (OneLake / ADLS Gen2) 파일 동기화 가이드

> 최신 검증 버전: **Hadoop 3.4.1** + **hadoop-azure 3.4.1** + **azure-storage 8.6.6**
> 대상 스토리지: **OneLake (Microsoft Fabric)** 또는 **ADLS Gen2 (HNS enabled)**
> 인증: **Entra ID Service Principal (OAuth 2.0 Client Credentials)**

---

## 1. Hadoop Optional Tools 등록

`hadoop-azure`, `hadoop-azure-datalake` 모듈을 classpath 에 추가합니다.

- 파일 위치: `/usr/local/hadoop/etc/hadoop/hadoop-env.sh`

```bash
export HADOOP_OPTIONAL_TOOLS="hadoop-azure,hadoop-azure-datalake"
```

> 위 한 줄만 등록해도 `share/hadoop/tools/lib/*` 의 jar 가 자동으로 로딩됩니다. 별도 jar 복사는 버전 충돌(특히 `jetty-util*`, `wildfly-openssl`) 을 유발할 수 있으므로 가능한 피하십시오.

---

## 2. 필수 라이브러리 (수동 설치가 필요한 경우)

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

## 3. OAuth 2.0 Client Credentials 설정

- 파일 위치: `/usr/local/hadoop/etc/hadoop/core-site.xml`
- 참조: https://hadoop.apache.org/docs/r3.4.1/hadoop-azure/abfs.html#OAuth_2.0_Client_Credentials

### 3-1. 단일 계정 전용 설정

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

### 3-2. 계정별 분기 설정 (여러 스토리지 / Fabric workspace 사용 시)

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

### 3-3. Secret 을 평문으로 두지 않기 — Hadoop Credential Provider

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

## 4. URI 규칙

| 대상 | URI 형식 |
| --- | --- |
| ADLS Gen2 | `abfss://<container>@<account>.dfs.core.windows.net/<path>` |
| OneLake (Fabric) | `abfss://<workspace>@onelake.dfs.fabric.microsoft.com/<lakehouse>.Lakehouse/<Files\|Tables>/<path>` |

> OneLake 의 workspace 이름에 공백 / 특수문자가 있으면 **GUID** 또는 **`__` (이중 언더스코어 prefix)** 형태의 정규화된 이름을 사용하세요. (예: `__WS_FabricWorkshop`)

---

## 5. Sample Hadoop Commands

### 5-1. 디렉터리 조회

```bash
# OneLake Files 영역 조회
hadoop fs -ls abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/Upload/

# 재귀 조회 + 사람이 읽기 좋은 사이즈
hadoop fs -ls -R -h abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/

# 용량 합계
hadoop fs -du -s -h abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/Upload/
```

### 5-2. 디렉터리 생성 / 삭제

```bash
hadoop fs -mkdir -p abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/Upload/2026/05/

# skipTrash 로 즉시 삭제 (OneLake 는 trash 미지원)
hadoop fs -rm -r -skipTrash abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/Upload/old/
```

### 5-3. 단일 파일 업로드 / 다운로드

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

### 5-4. DistCp — 대용량 / 디렉터리 전체 동기화

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

### 5-5. DistCp — 로컬 → OneLake 직접 전송

`hdfs://` 가 아닌 로컬 파일도 `file://` 스킴으로 distcp 사용 가능합니다.

```bash
hadoop distcp \
  -m 8 \
  file:///data/iso/enu_sql_server_2022_developer_edition_x64_dvd_7cacf733.iso \
  abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/Upload/
```

### 5-6. DistCp — 명령행에서 OAuth 자격증명 주입 (config 파일 없이)

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

### 5-7. DistCp — 클라우드 ↔ 클라우드 (ADLS Gen2 → OneLake)

```bash
hadoop distcp \
  -update -m 32 -bandwidth 100 -strategy dynamic \
  -numListstatusThreads 40 \
  abfss://raw@stadlsgen2prod.dfs.core.windows.net/curated/sales/ \
  abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/sales/
```

> 같은 SP 가 두 계정 모두에 접근 가능하면 위 한 줄이면 충분합니다. 계정별로 다른 SP 가 필요하면 **3-2** 의 account-scoped 설정을 사용하세요.

### 5-8. DistCp — Snapshot / 증분 전송 (HDFS 소스)

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

## 6. 동작 확인 / 트러블슈팅

### 6-1. 토큰 발급 검증

```bash
hadoop fs -ls abfss://__WS_FabricWorkshop@onelake.dfs.fabric.microsoft.com/Workshop_Lakehouse_Silver.Lakehouse/Files/ 2>&1 | tee /tmp/abfs.log
```

DEBUG 로그:

```bash
export HADOOP_ROOT_LOGGER=DEBUG,console
export HADOOP_OPTS="$HADOOP_OPTS -Dorg.apache.hadoop.fs.azurebfs=DEBUG"
hadoop fs -ls abfss://...
```

### 6-2. 자주 보는 오류

| 메시지 | 원인 | 조치 |
| --- | --- | --- |
| `AADToken: HTTP connection failed for getting token ... 401 Unauthorized` | client id / secret / tenant 오타, 또는 SP 가 만료된 secret 사용 | Entra ID 포털에서 secret 재발급 후 jceks 갱신 |
| `403 AuthorizationPermissionMismatch` | SP 에 RBAC/ACL 미부여 | Fabric workspace 에 **Contributor** 부여 또는 ADLS 에 **Storage Blob Data Contributor** 부여 |
| `FileSystem ... not found` (OneLake) | `fs.azure.createRemoteFileSystemDuringInitialization=true` 가 OneLake 에서 동작하지 않음 | Lakehouse 를 Fabric UI 에서 먼저 생성, 옵션은 `false` 로 |
| `No FileSystem for scheme: abfss` | hadoop-azure jar 미로딩 | `hadoop classpath` 확인, `HADOOP_OPTIONAL_TOOLS` 등록 여부 확인 |
| `SSLHandshakeException` | 사내 프록시/인증서 | `-Dhttps.proxyHost`/`-Dhttps.proxyPort` 또는 `fs.azure.ssl.channel.mode=Default` 로 변경 |

### 6-3. 권한 부여 예시

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

## 7. 운영 체크리스트

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
