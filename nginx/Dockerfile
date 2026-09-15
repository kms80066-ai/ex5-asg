# 1. (필수) 첫번째 위치, 기본 이미지 설정
FROM nginx:1.31.3-alpine

# RUN: 컨테이너(기본 이미지) 내부 명령
# default.conf 파일을 사용자의 파일로 교체하기 전 삭제
RUN rm /etc/nginx/conf.d/default.conf

# COPY: 컨테이너 내부로 파일 복사
# 3. default.conf 파일을 사용자의 파일로 교체
COPY ./default.conf /etc/nginx/conf.d/default.conf

# 4. HTML 내의 파일 및 디렉토리를 컨테이너로 복사, /표시로 디렉토리 정의
COPY ./html/ /usr/share/nginx/html/

# 5. 컨테이너 내부 포트 정의
EXPOSE 80

# 6. 컨테이너 시작할 때 실행할 명령 구성
CMD ["nginx","-g","daemon off;"]