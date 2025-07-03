# Uso de Bedrock para probar el modelo

1. Trabajar en `Estados Unidos` `Norte de Virginia` `us-east-1`

2. En la seccion `Areas de Juego` ir a `Chat / Text`

3. Dar click en `Selccionar el modelo`

4. En `Amazon`, seleccionar `Nova Pro 1.0`, y en `Inferencia`, seleccionar `Bajo demanda`, dar `Aplicar`.

5. Probar el modelo con el siguiente prompt: `¿Cuál es la capital de Perú?`

# Creacion de una Lambda para que use nova pro

1. Crear la función Lambda:

    - Ve a la consola AWS → Lambda → crea función en Python 3.13

2. Usar el código de prueba:

```python
import json
import boto3

client = boto3.client("bedrock-runtime", region_name="us-east-1")

def lambda_handler(event, context):
    prompt = event.get("prompt", "Texto de prueba")
    response = client.converse(
        modelId="amazon.nova-pro-v1:0",
        messages=[{"role": "user", "content": [{"text": prompt}]}]
    )
    # Extraemos la respuesta correctamente
    msg = response["output"]["message"]
    text = msg["content"][0]["text"]
    return {"statusCode": 200, "body": text}

```

Guardarlo y desplegarlo.


3. En la consola Lambda, ve a **Test** → crea evento:

```json
{ "prompt": "¿Cuál es la capital de Perú?" }
```

4. Ejecuta. Si responde con una frase coherente (“Lima es la capital de Perú”), ¡funciona perfectamente!

<br>
En caso de que salga un error de `AccessDeniedException`, similar a: 

        Response:
        {
        "errorMessage": "An error occurred (AccessDeniedException)
    

Se debe Crear/Adjuntar la política IAM correcta

1. Entra al servicio **IAM → Politicas → Crear politica → cambiarlo a JSON** y pega:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowBedrockInvokeNovaPro",
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "arn:aws:bedrock:us-east-1::foundation-model/amazon.nova-pro-v1:0"
    },
    {
      "Sid": "AllowListModels",
      "Effect": "Allow",
      "Action": "bedrock:ListFoundationModels",
      "Resource": "*"
    }
  ]
}
```

Este fragmento da permiso para invocar precisamente tu modelo y también listar modelos.

Después, agregar un nombre a la politica y agregarlo.


 Reintenta la ejecución de la Lambda

1. Ve a **Lambda → tu función**.
2. Ejecuta de nuevo el test con el mismo evento:

```json
{ "prompt": "¿Cuál es la capital de Perú?" }
```

---
# Bedrock + Knowledge Bases

1. Subir los documentos a S3 (PDF, Word, texto…).

2. AWS extrae automáticamente el contenido, genera embeddings y los indexa.

3. Cuando se hace una pregunta, busca los fragmentos relevantes y los envía junto con el prompt a Nova Pro.

4. Modificar los permisos necesarios para que la Lambda pueda acceder a la Knowledge Base y a bedrock en IAM.

## Crear un bucket en S3 y subir documentos

1. Entra a la **consola AWS** → servicio **S3** → clic en **“Create bucket”**.

   * Nombre ejemplo: `mi-kb-bucket`
   * Región: **us-east-1** (debe coincidir con Bedrock)
   * Tipo de bucket: `Uso general`

2. Abre el bucket → **Upload** → Cargar archivos

3. Presiona **Upload** y espera a que los archivos estén listos.

## Crear Knowledge Base en Bedrock

1. Buscar **Knowledge bases** de Bedrock (en us‑east‑1) → luego **Create** → selecciona **Crear base de conocimientos con almacén vectorial**.

2. Completa el formulario:

   * Name: `mi-kb`
   * Data source: **Amazon S3**
   * Indica el bucket `mi-kb-bucket` y selecciona los archivos
   * Embeddings model: **amazon.titan-embed-text-v2**
   * Vector store: selecciona “Quick create new” (usará OpenSearch Serverless)

3. Haz clic en **Create**. Espera hasta que el estado diga **Available** (sincroniza, indexa).

## Probar la KB desde la consola

1. Dentro de tu KB (`mi-kb`), haz clic en **Test knowledge base**.
2. Activa la opción **“Generate responses”**.
3. Escribe:

   ```
   ¿Que es PEP?
   ```

- En IAM, agregar la politica que se genero en el paso anterior.

```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "bedrock:ListFoundationModels",
          "bedrock:InvokeModel"
        ],
        "Resource": [
          "arn:aws:bedrock:us-east-1::foundation-model/amazon.titan-embed-text-v2:0",
          "arn:aws:bedrock:us-east-1::foundation-model/amazon.titan-embed-text-v1:0"
        ]
      },
      {
        "Effect": "Allow",
        "Action": [
          "s3:GetObject",
          "s3:ListBucket"
        ],
        "Resource": [
          "arn:aws:s3:::prueba2-nova-pro",
          "arn:aws:s3:::prueba2-nova-pro/*"
        ]
      }
    ]
  }
```


---

## Invocar la KB desde código (Lambda)

Usaremos la API `retrieve_and_generate`:

```python
import boto3
import json
import logging
import time

# Configura logging mínimo
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger()

def lambda_handler(event, context):
    # Inicializa cliente de Bedrock en us-east-1
    client = boto3.client('bedrock-agent-runtime', region_name='us-east-1')

    # Obtener consulta del evento o usar una por defecto
    user_query = event.get('query', '¿Que son las expresiones?')
    
    # Validar entrada
    if not user_query or not isinstance(user_query, str):
        logger.error("Consulta inválida o vacía")
        return {
            'statusCode': 400,
            'body': json.dumps({'error': 'Consulta inválida o vacía'})
        }

    try:
        # Registrar inicio de la llamada
        start_time = time.time()
        
        # Llamada optimizada a Bedrock
        response = client.retrieve_and_generate(
            input={'text': user_query},
            retrieveAndGenerateConfiguration={
                'type': 'KNOWLEDGE_BASE',
                'knowledgeBaseConfiguration': {
                    'knowledgeBaseId': 'AVDJ3M69B7',  # Verifica que este ID sea correcto
                    'modelArn': 'arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-sonnet-20240229-v1:0',
                    'retrievalConfiguration': {
                        'vectorSearchConfiguration': {
                            'numberOfResults': 1  # Reducido para mejorar rendimiento
                        }
                    }
                }
            }
        )

        # Registrar tiempo de respuesta
        logger.info(f"Tiempo de llamada a Bedrock: {time.time() - start_time:.2f} segundos")

        # Extraer respuesta
        output_text = response.get('output', {}).get('text', 'No se encontró respuesta.')
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'respuesta': output_text
            }, ensure_ascii=False)  # Soporte para caracteres en español
        }

    except client.exceptions.ValidationException as ve:
        logger.error(f"ValidationException: {str(ve)}")
        return {
            'statusCode': 403,
            'body': json.dumps({'error': f'ValidationException: {str(ve)}'})
        }
    except client.exceptions.AccessDeniedException as ade:
        logger.error(f"AccessDeniedException: {str(ade)}")
        return {
            'statusCode': 403,
            'body': json.dumps({'error': f'AccessDeniedException: {str(ade)}'})
        }
    except client.exceptions.ThrottlingException as te:
        logger.error(f"ThrottlingException: {str(te)}")
        return {
            'statusCode': 429,
            'body': json.dumps({'error': 'Límite de solicitudes excedido'})
        }
    except Exception as e:
        logger.error(f"Error general: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
   ```
3. Ejecuta un test con el evento:

   ```json
   {}
   ```

---

## ✅ Resumen rápido

| Etapa      | Qué haces                                       |
| ---------- | ----------------------------------------------- |
| 1. S3      | Subes documentos `.txt` con info.               |
| 2. Bedrock | Creas KB conectada a tu bucket.                 |
| 3. Consola | Pruebas la KB con una pregunta.                 |
| 4. Lambda  | Escribes código con `retrieve_and_generate`.    |
| 5. Test    | Compruebas que la Lambda devuelve la respuesta. |

---

Con esto, **ya usas tu propia información** en Nova Pro sin nada complicado ni reentrenamiento 💡. Avísame si necesitas que te dé los permisos IAM exactos, o cómo generar el ID de la KB con AWS CLI.

[1]: https://docs.aws.amazon.com/code-library/latest/ug/python_3_bedrock-runtime_code_examples.html?utm_source=chatgpt.com "Amazon Bedrock Runtime examples using SDK for Python (Boto3)"
[2]: https://docs.aws.amazon.com/bedrock/latest/userguide/kb-test-retrieve-generate.html?utm_source=chatgpt.com "Query a knowledge base and generate responses based off the ..."
[3]: https://john-tucker.medium.com/amazon-bedrock-knowledge-bases-by-example-8686109ac5b1?utm_source=chatgpt.com "Amazon Bedrock Knowledge Bases by Example | by John Tucker"
[4]: https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html?utm_source=chatgpt.com "Retrieve data and generate AI responses with Amazon Bedrock ..."
[5]: https://docs.aws.amazon.com/bedrock/latest/APIReference/API_agent-runtime_RetrieveAndGenerate.html?utm_source=chatgpt.com "RetrieveAndGenerate - Amazon Bedrock - AWS Documentation"


Claro, puedes cargar un PDF de prueba en una Knowledge Base de Amazon Bedrock, y usarlo directamente para hacer preguntas. Aquí tienes cómo hacerlo, paso a paso y muy claro:

---

## 📄 Paso 1: Prepara el PDF

Crea un PDF simple con contenido como:

```
Título: Política de Devoluciones

Puedes devolver productos dentro de los 30 días. Deben estar en su embalaje original. El reembolso se hace en la misma forma de pago.
```

Guárdalo como, por ejemplo, `politica_devoluciones.pdf`.

---

## Paso 2: Crea o usa tu bucket de S3

1. En la consola de **S3**, crea un bucket en **us‑east‑1** (mismo region que Bedrock).
2. Súbelo desde tu PC: selecciona **Upload**, añade `politica_devoluciones.pdf`, y presiona **Upload**.

---

## Paso 3: Crea la Knowledge Base conectada a S3

1. Ve a **Amazon Bedrock** → **Knowledge bases** → clic en **Create** → elige **Vector store**.
2. Ponle un nombre (por ejemplo: `kb-pdf`).
3. Como data source, selecciona **Amazon S3**, y apunta al bucket + tu `politica_devoluciones.pdf`.
4. Selecciona el modelo de embeddings: `amazon.titan-embed-text-v2`.
5. Para vector store, elige “Quick create new” (usará OpenSearch Serverless).
6. Haz clic en **Create** y espera a que el estado sea **Available** ([docs.aws.amazon.com][1], [stackoverflow.com][2], [stackoverflow.com][3], [dev.to][4]).

---

## Paso 4: Prueba “Chat with your document” (alternativa rápida)

Puedes probar sin preparar la KB completa:

1. En **Knowledge bases**, hay una pestaña “Chat with your document”.
2. Sube directamente tu PDF desde la computadora.
3. Selecciona un modelo (Anthropic Sonnet o Nova Pro).
4. Escribe una pregunta, por ejemplo:

   ```
   ¿Cuál es la política de devoluciones?
   ```
5. Presiona **Run**, y la respuesta aparece junto con los fragmentos que respaldan la respuesta ([docs.aws.amazon.com][5], [john-tucker.medium.com][6]).

También te dejo este video paso a paso:

[Building a Knowledge Base for Amazon Bedrock Using a PDF File](https://www.youtube.com/watch?v=sOuScuZK08k&utm_source=chatgpt.com)

---

## Paso 5: Usar el PDF a través de la KB en Lambda

Si creaste la KB conectada al bucket, luego sincronizala y usa este código en Lambda:

```python
import json
import boto3

bk = boto3.client("bedrock-agent-runtime", region_name="us-east-1")

def lambda_handler(event, context):
    pregunta = event.get("question", "¿Cuál es la política de devoluciones?")
    resp = bk.retrieve_and_generate(
        input={"text": pregunta},
        retrieveAndGenerateConfiguration={
            "type": "EXTERNAL_SOURCES",
            "externalSourcesConfiguration": {
                "modelArn": "arn:aws:bedrock:us-east-1::foundation-model/amazon.nova-pro-v1:0",
                "sources": [
                    {
                      "sourceType": "S3",
                      "s3Location": {"uri": "s3://tu-bucket/politica_devoluciones.pdf"}
                    }
                ]
            }
        }
    )
    return {"statusCode": 200, "body": resp["generation"]["text"]}
```

* Reemplaza `"tu-bucket"` por el nombre de tu bucket.
* Esto le dice a Nova Pro: usa el PDF del bucket como contexto para responder.
* Si todo está bien, la respuesta incluirá texto basado en el PDF ([dev.to][4]).

---

## ✅ Resumen

| Etapa                | Qué hacer                         |
| -------------------- | --------------------------------- |
| 1. Crear PDF         | Con tu política de devoluciones.  |
| 2. Subir a S3        | En bucket en us‑east‑1.           |
| 3. KB en Bedrock     | Conecta a S3 + PDF.               |
| 4. Probar en consola | “Chat with your document”.        |
| 5. Lambda + retrieve | Apunta a PDF en S3 en tu función. |

Con esto ya podrás **hacer preguntas basadas en tu PDF de prueba** y ver cómo Nova Pro responde con tu propio contenido. ¿Te ayudo con los permisos IAM o quieres que te dé la función lista para copiar y pegar?

[1]: https://docs.aws.amazon.com/bedrock/latest/userguide/kb-direct-ingestion-add.html?utm_source=chatgpt.com "Ingest documents directly into a knowledge base - Amazon Bedrock"
[2]: https://stackoverflow.com/questions/79422187/sending-an-attachment-to-claude-in-aws-bedrock?utm_source=chatgpt.com "Sending an attachment to Claude in AWS Bedrock - Stack Overflow"
[3]: https://stackoverflow.com/questions/78912309/aws-bedrock-rag-unable-to-sync-data-source-in-a-knowledge-base?utm_source=chatgpt.com "AWS Bedrock RAG - Unable to sync data source in a knowledge base"
[4]: https://dev.to/aws-builders/using-a-knowledge-base-to-connect-amazon-bedrock-to-your-custom-data-2gd9?utm_source=chatgpt.com "Configuring an Amazon Bedrock Knowledge Base - DEV Community"
[5]: https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base-chatdoc.html?utm_source=chatgpt.com "Chat with your document without a knowledge base configured"
[6]: https://john-tucker.medium.com/amazon-bedrock-knowledge-bases-by-example-8686109ac5b1?utm_source=chatgpt.com "Amazon Bedrock Knowledge Bases by Example | by John Tucker"
