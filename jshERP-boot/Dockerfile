FROM openjdk:8-jre-slim

WORKDIR /app

# COPY 路径相对于 docker-compose build context（项目根目录）
COPY jshERP-boot/target/jshERP.jar /app/jshERP.jar

RUN mkdir -p /opt/jshERP/{upload,export,logs}

EXPOSE 9999

ENTRYPOINT ["java", "-server", \
    "-Xms256m", "-Xmx512m", \
    "-Dlogs.home=/opt/jshERP/logs", \
    "-Dspring.config.location=file:/app/config/application.yml", \
    "-Dfile.encoding=UTF-8", \
    "-jar", "/app/jshERP.jar"]
