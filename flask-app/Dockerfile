FROM alpine:3.21.0

RUN apk add --no-cache build-base libffi-dev openssl-dev py3-pip python3

WORKDIR /usr/src/app

RUN python3 -m venv venv

ENV PATH="/usr/src/app/venv/bin:$PATH"

COPY requirements.txt ./
RUN pip install --no-cache-dir --upgrade pip && pip install --no-cache-dir -r requirements.txt

COPY app.py ./
COPY templates/index.html ./templates/

EXPOSE 5000

CMD ["python", "/usr/src/app/app.py"]

