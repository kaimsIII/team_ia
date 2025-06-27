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

3. Ahora debería responder sin error.

