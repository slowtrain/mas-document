ARG BUNDLE=all

##############################################################################
# 1단계 - Admin 이미지를 빌더로 사용하여 EAR 생성
##############################################################################

FROM manage-admin-dev:latest AS admin

ARG BUNDLE

USER root


##############################################################################
# 커스터마이징 적용
#
# Build Context:
#
#   manage/
#   ├─ applications/
#   └─ manage-developer/
#      └─ Dockerfile
#
##############################################################################

ADD applications /opt/IBM/SMP/writeable/maximo/applications

WORKDIR /opt/IBM/SMP/writeable/maximo/applications/maximo

RUN ls -l


##############################################################################
# Local Form Authentication용 Deployment Descriptor 적용
##############################################################################


##############################################################################
# maximo-all / maximouiweb
##############################################################################

WORKDIR /opt/IBM/SMP/writeable/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-all/maximouiweb/webmodule/WEB-INF/

RUN mv web.xml web-oidc.xml && \
    mv web-dev.xml web.xml


##############################################################################
# maximo-all / maxrestweb
##############################################################################

WORKDIR /opt/IBM/SMP/writeable/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-all/maxrestweb/webmodule/WEB-INF/

RUN mv web.xml web-oidc.xml && \
    mv web-dev.xml web.xml


##############################################################################
# maximo-all / mboweb
##############################################################################

WORKDIR /opt/IBM/SMP/writeable/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-all/mboweb/webmodule/WEB-INF/

RUN mv web.xml web-oidc.xml && \
    mv web-dev.xml web.xml


##############################################################################
# maximo-all / meaweb
##############################################################################

WORKDIR /opt/IBM/SMP/writeable/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-all/meaweb/webmodule/WEB-INF/

RUN mv web.xml web-oidc.xml && \
    mv web-dev.xml web.xml


##############################################################################
# maximo-foundation / mboweb
##############################################################################

WORKDIR /opt/IBM/SMP/writeable/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-foundation/mboweb/webmodule/WEB-INF/

RUN mv web.xml web-oidc.xml && \
    mv web-dev.xml web.xml


##############################################################################
# maximo-foundation / meaweb
##############################################################################

WORKDIR /opt/IBM/SMP/writeable/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-foundation/meaweb/webmodule/WEB-INF/

RUN mv web.xml web-oidc.xml && \
    mv web-dev.xml web.xml


##############################################################################
# maximo-cron
##############################################################################

WORKDIR /opt/IBM/SMP/writeable/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-cron/webmodule/WEB-INF/

RUN mv web.xml web-oidc.xml && \
    mv web-dev.xml web.xml


##############################################################################
# maximo-mea / maxrestweb
##############################################################################

WORKDIR /opt/IBM/SMP/writeable/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-mea/maxrestweb/webmodule/WEB-INF/

RUN mv web.xml web-oidc.xml && \
    mv web-dev.xml web.xml


##############################################################################
# maximo-mea / meaweb
##############################################################################

WORKDIR /opt/IBM/SMP/writeable/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-mea/meaweb/webmodule/WEB-INF/

RUN mv web.xml web-oidc.xml && \
    mv web-dev.xml web.xml


##############################################################################
# maximo-report
##############################################################################

WORKDIR /opt/IBM/SMP/writeable/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-report/webmodule/WEB-INF/

RUN mv web.xml web-oidc.xml && \
    mv web-dev.xml web.xml


##############################################################################
# maximo-ui
##############################################################################

WORKDIR /opt/IBM/SMP/writeable/maximo/deployment/was-liberty-default/config-deployment-descriptors/maximo-ui/webmodule/WEB-INF/

RUN mv web.xml web-oidc.xml && \
    mv web-dev.xml web.xml


##############################################################################
# EAR 생성
##############################################################################

WORKDIR /opt/IBM/SMP/writeable/maximo/deployment/was-liberty-default

RUN ./maximo-${BUNDLE}.sh


##############################################################################
# 생성된 EAR 확인
##############################################################################

WORKDIR /opt/IBM/SMP/writeable/maximo/deployment/was-liberty-default/deployment/maximo-${BUNDLE}/maximo-${BUNDLE}-server/apps/

RUN ls -lh


##############################################################################
# 2단계 - ServerBundle 이미지를 기반으로 최종 개발 이미지 생성
##############################################################################

FROM manage-${BUNDLE}-dev:latest

ARG BUNDLE

USER root


##############################################################################
# 기존 ServerBundle EAR 확인
##############################################################################

RUN ls -lh /config/apps


##############################################################################
# Admin Builder에서 새로 생성한 EAR 복사
##############################################################################

COPY --from=admin \
    /opt/IBM/SMP/writeable/maximo/deployment/was-liberty-default/deployment/maximo-${BUNDLE}/maximo-${BUNDLE}-server/apps/*.ear \
    /config/apps/

RUN ls -lh /config/apps


##############################################################################
# 로컬 개발용 Liberty 설정
#
# MAS/OIDC용 server.xml 대신
# ServerBundle에 포함된 server-dev.xml 사용
##############################################################################

WORKDIR /opt/ibm/wlp/usr/servers/defaultServer

RUN mv server.xml server-oidc.xml && \
    mv server-dev.xml server.xml

WORKDIR /


##############################################################################
# Database
##############################################################################

ENV DB_SSL_ENABLED=nossl

ENV MXE_DB_URL='jdbc:db2://192.168.2.211:31899/BLUDB'
ENV MXE_DB_SCHEMAOWNER=maximo
ENV MXE_DB_DRIVER='com.ibm.db2.jcc.DB2Driver'

ENV MXE_DB_USER='db2inst1'
ENV MXE_DB_PASSWORD='X5D4gNDiRlMsPUu'


##############################################################################
# Maximo Encryption Keys
#
# 기존 MAS Manage DB와 동일한 encryption key 사용
##############################################################################



ENV MXE_SECURITY_OLD_CRYPTO_KEY='IAshvBJAkPArfYMRcgKPKDEu'
ENV MXE_SECURITY_CRYPTO_KEY='IAshvBJAkPArfYMRcgKPKDEu'
ENV MXE_SECURITY_OLD_CRYPTOX_KEY='dceVSktsCyUZWOiEpEXmChpb'
ENV MXE_SECURITY_CRYPTOX_KEY='dceVSktsCyUZWOiEpEXmChpb'


##############################################################################
# 로컬 개발 모드
#
# MAS SSO 대신 Maximo 자체 로그인 사용
##############################################################################

ENV MXE_USEAPPSERVERSECURITY=0
ENV MXE_MASDEPLOYED=0


##############################################################################
# Locale / Timezone
##############################################################################

ENV LC_ALL=en_US.UTF-8
ENV TZ=Asia/Seoul


##############################################################################
# Maximo Runtime User
##############################################################################

USER 1001