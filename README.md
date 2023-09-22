# SMSChatBot
This code allows you to respond to a text message using Twilio's API. The intended use is to make an SMS-enabled chatbot, as outlined in my Medium blog post.
The code looks like this:

# Begin Code
import os
import io
import boto3
import json
import base64
from urllib import parse
import re
from twilio.rest import Client

TWILIO_SMS_URL = "https://api.twilio.com/2010-04-01/Accounts/{}/Messages.json"
TWILIO_ACCOUNT_SID = os.environ.get("TWILIO_ACCOUNT_SID")
TWILIO_AUTH_TOKEN = os.environ.get("TWILIO_AUTH_TOKEN")
FROM_NUMBER= os.environ.get("FROM_NUMBER")

# grab environment variables
ENDPOINT_NAME = os.environ['ENDPOINT_NAME']
runtime= boto3.client('runtime.sagemaker')

def lambda_handler(event, context):
    print(event)
    
    # Decode body
    body_from_receiving_text = base64.b64decode(event['body'].encode('ascii'))
    body_from_receiving_text = body_from_receiving_text.decode('ascii')
    print(body_from_receiving_text)
    
    # grab the number it was sent from
    number_received_from = re.search('From=%2B(.*)&ApiVersion', body_from_receiving_text)
    print(number_received_from.group(1))
    
    # Grab the message sent
    message_received = re.search('&Body=(.*)&FromCountry', body_from_receiving_text)
    print(message_received.group(1))
    
    # Call the LLM
    response = runtime.invoke_endpoint(EndpointName=ENDPOINT_NAME,
                                        ContentType='application/json',
                                        Body=json.dumps({
                                         "inputs": [
                                          [
                                           {"role": "system", "content": "You are chat bot who answers questions about general knowledge"},
                                           {"role": "user", "content": message_received.group(1)}
                                          ]
                                         ],
                                         "parameters": {"max_new_tokens":256, "top_p":0.9, "temperature":0.6}
                                        }),
                                      CustomAttributes="accept_eula=true")
    print(response)
    
    result = json.loads(response['Body'].read().decode())
    print(result)
    
    # Params for text to be sent
    to_number = '+' + number_received_from.group(1)
    from_number = '+1' + FROM_NUMBER
    body = result[0]["generation"]["content"]

    if not TWILIO_ACCOUNT_SID:
        return "Unable to access Twilio Account SID."
    elif not TWILIO_AUTH_TOKEN:
        return "Unable to access Twilio Auth Token."
    elif not to_number:
        return "The function needs a 'To' number in the format +12023351493"
    elif not from_number:
        return "The function needs a 'From' number in the format +19732644156"
    elif not body:
        return "The function needs a 'Body' message to send."

    client = Client(TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN)

    message = client.messages \
    .create(
         body=body,
         from_=from_number,
         to=to_number
     )
    
    return (message.sid)

