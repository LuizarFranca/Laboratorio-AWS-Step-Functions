# Laboratorio-AWS-Step-Functions
Laboratório prático para consolidar workflows automatizados usando AWS Step Functions.


1. Objetivo
Demonstrar a criação e automação de um fluxo de trabalho (workflow) utilizando AWS Step Functions, integrando serviços como AWS Lambda, S3, DynamoDB e SNS.
O laboratório visa consolidar o entendimento de orquestração, monitoramento e tratamento de exceções em fluxos automatizados.

2. Pré-requisitos

Conta AWS com permissões de administrador.
 - AWS CLI instalado e configurado.
 - Conhecimentos básicos em: AWS Lambda, IAM (permissões e roles),CloudWatch
 - Editor de código (Visual Studio Code recomendado).
 - Ferramentas online para diagramação (Draw.io, Lucidchart ou equivalente).

3. Arquitetura do Laboratório

O workflow irá:
 - Detectar o upload de um arquivo no S3.
 - Executar uma função Lambda de validação.
 - Processar os dados via Lambda de processamento.
 - Registrar resultados no DynamoDB.
- Enviar notificação SNS ao final da execução.


Diagrama Conceitual
vbnet
S3 → Step Function → Lambda Validate → Lambda Process → DynamoDB → SNS

4. Etapas do Laboratório

Etapa 1: Criar recursos base
  1.Criar um bucket S3
bash

aws s3 mb s3://lab-stepfunctions-luiza


2.Criar uma tabela DynamoDB
bash

aws dynamodb create-table \
  --table-name ProcessedFiles \
  --attribute-definitions AttributeName=FileName,AttributeType=S \
  --key-schema AttributeName=FileName,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST



Lambda 2 – Processamento
Arquivo: lambda_process.py
python

import json

def lambda_handler(event, context):
    file_name = event['file_name']
    # Simulação de processamento
    print(f"Processando arquivo: {file_name}")
    return {'status': 'processed', 'file_name': file_name, 'result': 'success'}


Lambda 3 – Armazenamento
Arquivo: lambda_store.py
python

import boto3
import json

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('ProcessedFiles')

def lambda_handler(event, context):
    file_name = event['file_name']
    result = event['result']
    table.put_item(Item={'FileName': file_name, 'Status': result})
    return {'status': 'stored', 'file_name': file_name}

Lambda 4 – Notificação
Arquivo: lambda_notify.py
python

import boto3

sns = boto3.client('sns')
TOPIC_ARN = 'arn:aws:sns:REGION:ACCOUNT_ID:WorkflowNotification'

def lambda_handler(event, context):
    sns.publish(
        TopicArn=TOPIC_ARN,
        Message=f"Arquivo {event['file_name']} processado com sucesso!",
        Subject='Notificação de Workflow'
    )
    return {'status': 'notified'}

Etapa 3: Criar o Tópico SNS
bash
aws sns create-topic --name WorkflowNotification

Anote o ARN retornado e atualize na Lambda lambda_notify.py.

Etapa 4: Criar a State Machine
No console da AWS Step Functions:
 1.Crie uma State Machine (Standard).
 2.Escolha “Author with code snippets”.
 3.Cole a definição JSON abaixo:
json

{
  "Comment": "Workflow de processamento de arquivos",
  "StartAt": "ValidateFile",
  "States": {
    "ValidateFile": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:lambda_validate",
      "Next": "ProcessFile",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "FailState"
      }]
    },
    "ProcessFile": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:lambda_process",
      "Next": "StoreResult"
    },
    "StoreResult": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:lambda_store",
      "Next": "Notify"
    },
    "Notify": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:lambda_notify",
      "End": true
    },
    "FailState": {
      "Type": "Fail",
      "Cause": "Erro na validação do arquivo"
    }
  }
}

Etapa 5: Integrar com S3
Crie uma Lambda simples que, ao detectar o upload de um arquivo, inicia a execução da Step Function.
python

import boto3
import json

sfn = boto3.client('stepfunctions')

STATE_MACHINE_ARN = 'arn:aws:states:REGION:ACCOUNT_ID:stateMachine:WorkflowProcessamento'

def lambda_handler(event, context):
    file_name = event['Records'][0]['s3']['object']['key']
    sfn.start_execution(
        stateMachineArn=STATE_MACHINE_ARN,
        input=json.dumps({'file_name': file_name})
    )

Depois, configure o S3 Event Trigger para essa Lambda (evento: ObjectCreated).

5. Testes e Validação
  1.Envie um arquivo .csv ao bucket S3.
  2.Acompanhe a execução no AWS Step Functions Console.
  3.Verifique logs no CloudWatch.
  4.Confirme registro no DynamoDB e notificação SNS.

6. Resultados Esperados
  -Execução bem-sucedida da State Machine.
  -Logs detalhados de cada estado.
  -Registro no DynamoDB com status “success”.
  -Notificação SNS recebida.

7. Extensões do Laboratório
  -Adicionar estados de paralelismo (Parallel State).
  -Implementar retries automáticos.
  -Adicionar Step Functions Express para workflows de alta frequência.
  -Integrar com AWS EventBridge para eventos complexos.

8. Conclusão
 Este laboratório permite consolidar os conceitos de orquestração e automação usando AWS Step Functions, 
fornecendo base prática para criação de pipelines corporativos resilientes e auditáveis.
