FROM public.ecr.aws/lambda/python:3.10
COPY app.py ${LAMBDA_TASK_ROOT}
COPY requirements.txt  .
RUN  pip3 install -r requirements.txt
RUN  apk upgrade openssl
CMD ["app.handler"]
