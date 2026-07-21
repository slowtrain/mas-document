
###############################################################################

# 1. OpenShift 콘솔에서 토큰 발급

# OpenShift 웹 콘솔

# → 우측 상단 사용자 이름

# → Copy login command

# → 인증 방식 선택

# → Display Token

# 발급된 sha256~... 토큰을 아래 read -s TOKEN 입력 후 붙여넣는다.

# 토큰은 채팅이나 문서에 저장하지 않는다.

###############################################################################

# Bastion

read -s TOKEN

oc login --token="$TOKEN" 
  --server="https://api.itz-cvysxp.infra01-lb.syd05.techzone.ibm.com:6443"

unset TOKEN

oc whoami
oc whoami --show-server

###############################################################################

# 2. Manage용 Db2 Pod 접속

###############################################################################

oc rsh -n db2u c-mas-inst1-gtmdemo-manage-db2u-0

###############################################################################

# 3. 아래 명령은 Db2 Pod 내부에서 실행

###############################################################################

bash -l
su - db2inst1
db2 connect to BLUDB

###############################################################################

# 4. 관리 모드 및 DB 구성 상태 확인

###############################################################################

db2 "select rtrim(varname) as varname,
            rtrim(varvalue) as varvalue
       from MAXIMO.MAXVARS
      where varname in ('ADMINRESTART','CONFIGURING')
      with ur"

###############################################################################

# 반드시 아래 상태인지 확인

# ADMINRESTART  ON

# CONFIGURING   0

# CONFIGURING=1이면 아래 UPDATE를 실행하지 않는다.

###############################################################################

db2 "update MAXIMO.MAXVARS
        set varvalue='OFF'
      where varname='ADMINRESTART'
        and varvalue='ON'"

db2 "commit"

###############################################################################

# 5. 변경 결과 확인

###############################################################################

db2 "select rtrim(varname) as varname,
            rtrim(varvalue) as varvalue
       from MAXIMO.MAXVARS
      where varname in ('ADMINRESTART','CONFIGURING')
      with ur"

###############################################################################

# 정상 결과

# ADMINRESTART  OFF

# CONFIGURING   0

###############################################################################

db2 terminate
exit
exit

###############################################################################

# 6. Bastion에서 OpenShift 로그아웃

###############################################################################

oc logout
