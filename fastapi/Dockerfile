FROM python:3.9-slim
# 1. 작업 디렉토리 정의
WORKDIR /app

# 2. 환경변수 설정
# PYTHONDONTWRITEBYTECODE: 파이썬이 .pyc 파일을 생성하지 않도록 설정
# PYTHONUNBUFFERED: 파이썬 출력 버퍼링을 비활성화하여 로그를 실시간으로 확인할 수 있도록 설정
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# 3. requirements.txt 파일을 컨테이너로 복사
# 명령 마지막에 있는 닷은 WORKDIR에 정의한 디렉토리를 의미 합니다.
COPY requirements.txt .

# 4. requirements.txt 파일에 정의된 패키지(라이브러리) 설치
# --no-cache-dir 옵션은 설치 후 캐시를 남기지 않도록 하는 옵션입니다.(용량 절약)
RUN pip install --no-cache-dir -r requirements.txt

# 5. 현재 디렉토리에 있는 모든 내용을 이미지로 복사
COPY ./app/ .

# 6. 컨테이너 내부 포트 정의
EXPOSE 8000

# 7. 컨테이너 시작할 때 실행할 명령 구성
# "--reload": 운영 환경에서는 포함하지 않습니다.(개발 환경에서만 사용)
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]


# docker run 예시로 nginx의 포트번호는 conf 파일로 돼있는 반면에 이것은 수정 가능
# CMD만 사용한 경우:CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
# 1) docker run -d -p 9000:9000 --name my-app my-fastapi-v1 uvicorn main:app --host 0.0.0.0 --port 9000

# ENTRYPOINT와 CMD를 함께 사용한 경우
# ENTRYPOINT ["uvicorn"]
# CMD ["main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
# 2) docker run -d -p 9000:9000 --name my-app my-fastapi-v1 --port 9000