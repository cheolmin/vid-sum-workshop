# etc) link

4-1 extractaudio



```python
// Some code

import json
import subprocess
import boto3
import urllib
import os

s3 = boto3.resource('s3')

def lambda_handler(event, context):
    print(event)
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'], encoding='utf-8')

    s3.Bucket(bucket).download_file(key, '/tmp/input.mp4')
    
    print('Audio File Info')
    subprocess.call(['ffmpeg', '-y', '-i', '/tmp/input.mp4', '/tmp/audio.mp3'])
    
    save_key = key.split('/')[-1]
    save_key = save_key.split('.')[0] + '.mp3'
    
    upload = s3.Bucket(bucket).put_object(Key='audios/'+save_key, Body=open('/tmp/audio.mp3', 'rb'))
    print(upload)
    
    return {
        'statusCode': 200,
        'body': event
    }
```

{% embed url="https://enm-share.s3.us-west-2.amazonaws.com/cj-extract-audio-e2ad4433-a1fa-48fd-a1aa-d4347d92f769.zip?response-content-disposition=inline&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEMz%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLWVhc3QtMSJIMEYCIQCxnocurDQij%2FGhhHdnLiKPoOXfXayfEceBuj7mWs6nBAIhAKhQIlMXtBIgnbvXpIPo%2FUvTD594Hv0iTIpRvf1pkBdTKukCCIX%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEQABoMODAxMzAzNDgyNTY1IgwqSKi951rtq6VaigsqvQIwsSdJB10B7xk9O22hXRqsdxWu%2BRcnecHyWdgqkcOtZvVs7p5NKhLBGFqRMGGSO4w2h7X6Xt3fBWGPYRs%2Bilv6%2BB0ZgyIiuPVSBegVTDpGKYytBZgrINr17faxSnSLICnNBqQRP5myQqmvLhU163xkcolcnOHSppYw2WQBgaNlFAStx9CV%2BvDPNcZ38ESWafMW0Loi0Nw3OnUlXY8pNMudX%2BeI06YIk7cUWC2GRC6tczZF4fp%2BIQSElMD0wTGVlZiDFF8q1LKIj1gH5K00DZzgiLt%2FxUCb6SaEmCYzEeNsWvf%2Fg5HOA%2BzvbYB%2BgLYWxmXSm0puXm9zcZ%2Fjw0DLkXqh4Gh7N94jngQP0LZzt%2FHwMG9%2B3xySB%2F%2BJglqu%2BnE7aXRydL5U%2FWm5pMKXO98r4Q1NWvfGEiPgK%2FQpa9%2FUMjDi35K0BjqGAo4vvliFOkt%2FCkEgsbdiCFWbEb79HHsx8kGYL4K%2B5CLWQ38FiCtMyFKd0Cx6xAV4nwoL1JrTNhfTMXEtnXsPYXNPS2vuSVkDGLC6n44cR9T0HB%2F8JZCrFvcQO2reeTn6PIUAYY%2F%2BNs3HtJ5SXNIBJp%2B5RWPQmv0BYDMwVkxWFGOr2%2FIRMwycdl34UH2mWvh1KN2n1V2BcV5ZkDwmIxVazUH88buQErhuT9e7gRa0LstXDIvnaRhB9HCDCPkgRKw%2FSEHCFVnIrB8c7Ltarw%2BGGYsTIUs%2Bysoc5dAUpzkZ7hjELv0dIJdZfeM7GGVDt1i8KmV0LTHrk8OJiASNpq5IJlM9Lw0NnAg%3D&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Date=20240703T063205Z&X-Amz-SignedHeaders=host&X-Amz-Expires=43200&X-Amz-Credential=ASIA3VELI2DCTDFO2THD%2F20240703%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Signature=687a54b23cc0c2a7e1efa87f64d02a8fca974faaa801a39605e31a795f528010" %}

4-2 transcribe

```python
// Some code

import json
import boto3
import urllib
import botocore
import logging
import time
import requests

logger = logging.getLogger(__name__)

transcribe_client = boto3.client("transcribe")
s3 = boto3.client('s3')

def get_job(job_name):
    
    try:
        response = transcribe_client.get_transcription_job(
            TranscriptionJobName=job_name
        )
        job = response["TranscriptionJob"]
        logger.info("Got job %s.", job["TranscriptionJobName"])
    except botocore.exceptions.ClientError:
        logger.exception("Couldn't get job %s.", job_name)
        return None
    else:
        return job
        
def transcribe_file(job_name, file_uri, transcribe_client):
    transcribe_client.start_transcription_job(
        TranscriptionJobName=job_name,
        Media={"MediaFileUri": file_uri},
        MediaFormat="mp4",
        LanguageCode="ko-KR",
    )

    max_tries = 60
    while max_tries > 0:
        max_tries -= 1
        job = transcribe_client.get_transcription_job(TranscriptionJobName=job_name)
        job_status = job["TranscriptionJob"]["TranscriptionJobStatus"]
        if job_status in ["COMPLETED", "FAILED"]:
            print(f"Job {job_name} is {job_status}.")
            if job_status == "COMPLETED":
                print(
                    f"Download the transcript from\n"
                    f"\t{job['TranscriptionJob']['Transcript']['TranscriptFileUri']}."
                )
            break
        else:
            print(f"Waiting for {job_name}. Current status is {job_status}.")
        time.sleep(10)
    #transcript_vocab_table = requests.get(
    #    job["Transcript"]["TranscriptFileUri"]
    #).json()
    return True

def run_transcribe(transcribe_job_name, file_uri, debug=False):
    if debug == False:
        if transcribe_job_name is not None:
            transcribe_result = transcribe_file(transcribe_job_name, file_uri, transcribe_client)
    job_simple = get_job(transcribe_job_name)
    transcript_simple = requests.get(
        job_simple["Transcript"]["TranscriptFileUri"]
    ).json()

    return transcript_simple["results"]  # ["transcripts"]
    
def upload_file_s3(bucket, file_name, file):
    s3 = boto3.client('s3')
    encode_file = json.dumps(file, indent=4, ensure_ascii=False)
    try:
        s3.put_object(Bucket=bucket, Key=file_name, Body=encode_file)
        return True
    except: 
        return False

def lambda_handler(event, context):
    # TODO implement
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'], encoding='utf-8')
    
    input_file = key.split('/')[-1]
    
    transcribe_job_name = 'job_' + input_file.split('.')[0]

    file_uri = 'https://s3.us-east-1.amazonaws.com/{}/{}'.format(bucket, key)
    if get_job(transcribe_job_name) is None:
        transcribe_result = run_transcribe(transcribe_job_name, file_uri, debug=False)
    else:
        transcribe_result = run_transcribe(transcribe_job_name, file_uri, debug=True)

    print(transcribe_result)
    
    save = upload_file_s3(bucket, 'transcribe/'+input_file.split('.')[0]+'.json', transcribe_result)
    
    print(save)
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }

```

{% embed url="https://enm-share.s3.us-west-2.amazonaws.com/cj-transcribe-job-47abfccd-ed84-48e3-99e9-0d6264af8374.zip?response-content-disposition=inline&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEMz%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLWVhc3QtMSJIMEYCIQCxnocurDQij%2FGhhHdnLiKPoOXfXayfEceBuj7mWs6nBAIhAKhQIlMXtBIgnbvXpIPo%2FUvTD594Hv0iTIpRvf1pkBdTKukCCIX%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEQABoMODAxMzAzNDgyNTY1IgwqSKi951rtq6VaigsqvQIwsSdJB10B7xk9O22hXRqsdxWu%2BRcnecHyWdgqkcOtZvVs7p5NKhLBGFqRMGGSO4w2h7X6Xt3fBWGPYRs%2Bilv6%2BB0ZgyIiuPVSBegVTDpGKYytBZgrINr17faxSnSLICnNBqQRP5myQqmvLhU163xkcolcnOHSppYw2WQBgaNlFAStx9CV%2BvDPNcZ38ESWafMW0Loi0Nw3OnUlXY8pNMudX%2BeI06YIk7cUWC2GRC6tczZF4fp%2BIQSElMD0wTGVlZiDFF8q1LKIj1gH5K00DZzgiLt%2FxUCb6SaEmCYzEeNsWvf%2Fg5HOA%2BzvbYB%2BgLYWxmXSm0puXm9zcZ%2Fjw0DLkXqh4Gh7N94jngQP0LZzt%2FHwMG9%2B3xySB%2F%2BJglqu%2BnE7aXRydL5U%2FWm5pMKXO98r4Q1NWvfGEiPgK%2FQpa9%2FUMjDi35K0BjqGAo4vvliFOkt%2FCkEgsbdiCFWbEb79HHsx8kGYL4K%2B5CLWQ38FiCtMyFKd0Cx6xAV4nwoL1JrTNhfTMXEtnXsPYXNPS2vuSVkDGLC6n44cR9T0HB%2F8JZCrFvcQO2reeTn6PIUAYY%2F%2BNs3HtJ5SXNIBJp%2B5RWPQmv0BYDMwVkxWFGOr2%2FIRMwycdl34UH2mWvh1KN2n1V2BcV5ZkDwmIxVazUH88buQErhuT9e7gRa0LstXDIvnaRhB9HCDCPkgRKw%2FSEHCFVnIrB8c7Ltarw%2BGGYsTIUs%2Bysoc5dAUpzkZ7hjELv0dIJdZfeM7GGVDt1i8KmV0LTHrk8OJiASNpq5IJlM9Lw0NnAg%3D&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Date=20240703T063404Z&X-Amz-SignedHeaders=host&X-Amz-Expires=43200&X-Amz-Credential=ASIA3VELI2DCTDFO2THD%2F20240703%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Signature=e32fa0bd9f87f5fbbbebe83ae2295b2adc7aeba1f3d77c8755fac727281e9ae6" %}

4-3 frameext and captioning

```python
// Some code

import json
import cv2
import urllib
import boto3
import base64

s3 = boto3.resource('s3')
sagemaker = boto3.client('sagemaker-runtime')
blip_endpoint = 'endpoint-blip2-flan-t5-xl-2024-07-02-17-32-50-384'
SAMPLE_EVERY_SEC = 10

def encode_image(img):
    #with open(img_file, "rb") as image_file:

    #retval, buffer = cv2.imencode('.jpg', img)
    #jpg_as_text = base64.b64encode(buffer)
    string = base64.b64encode(cv2.imencode('.jpg', img)[1]).decode()

    #img_str = base64.b64encode(img)
    #base64_string = img_str.decode("latin1")
    return string

def run_inference(endpoint_name, inputs):
    response = sagemaker.invoke_endpoint(
        EndpointName=endpoint_name, Body=json.dumps(inputs)
    )
    return response["Body"].read().decode('utf-8')

def extract_frames(bucket, key, video_path, SAMPLE_EVERY_SEC=2, enable = True, method = ""):
    frames = []
    inference_results = []
    prompt = "Question: What scene is this picture? Answer:"
    save_path = key.split('/')[-1]
    save_path = save_path.split('.')[0]
    
    cap = cv2.VideoCapture(video_path)

    n_frames = cap.get(cv2.CAP_PROP_FRAME_COUNT)
    print(cv2.CAP_PROP_FPS)
    fps = cap.get(cv2.CAP_PROP_FPS)

    video_len = n_frames / fps

    print(f'Video length {video_len:.2f} seconds!')
    last_collected = -1

    if enable:
        cnt = 0
        while cap.isOpened():
            ret, frame = cap.read()
            if not ret:
                break

            timestamp = cap.get(cv2.CAP_PROP_POS_MSEC)
            second = timestamp // 1000

            if second % SAMPLE_EVERY_SEC == 0 and second != last_collected:
                last_collected = second
                # cv2.imwrite('/tmp/' + str(cnt) + '.png', frame)
                
                #img_string = cv2.imencode('.jpg', frame)[1].tostring()
                #upload = s3.Bucket(bucket).put_object(Key='images/'+ save_path + str(cnt)+'.png', Body=img_string)
                #print(upload, ' : images/'+ save_path + str(cnt)+'.png')
                
                b64_string = encode_image(frame)
                input_payload = {"prompt": prompt, "image":b64_string}
                result = run_inference(blip_endpoint, input_payload)
                
                inference_results.append(result)
                cnt += 1
                if cnt % 50 == 0 :
                    print(cnt , ' / ' , video_len)
    return inference_results

def lambda_handler(event, context):
    # TODO implement
    print(event['requestPayload']['Records'][0]['s3'])
    bucket = event['requestPayload']['Records'][0]['s3']['bucket']['name']
    key = urllib.parse.unquote_plus(event['requestPayload']['Records'][0]['s3']['object']['key'], encoding='utf-8')
    
    print(bucket)
    print(key)
    s3.Bucket(bucket).download_file(key, '/tmp/input.mp4')
    
    
    results = extract_frames(bucket, key, '/tmp/input.mp4', SAMPLE_EVERY_SEC, True)
    
    response = {
        'statusCode': 200,
        'bucket':bucket,
        'key':key,
        'body': results,
        'interval': SAMPLE_EVERY_SEC
        }
        
    print(response)
    return response

```

{% embed url="https://enm-share.s3.us-west-2.amazonaws.com/extractFrameAndCaptioning-a0aff59b-ba93-46ee-8d33-a737ae4a6ad5.zip?response-content-disposition=inline&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEMz%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLWVhc3QtMSJIMEYCIQCxnocurDQij%2FGhhHdnLiKPoOXfXayfEceBuj7mWs6nBAIhAKhQIlMXtBIgnbvXpIPo%2FUvTD594Hv0iTIpRvf1pkBdTKukCCIX%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEQABoMODAxMzAzNDgyNTY1IgwqSKi951rtq6VaigsqvQIwsSdJB10B7xk9O22hXRqsdxWu%2BRcnecHyWdgqkcOtZvVs7p5NKhLBGFqRMGGSO4w2h7X6Xt3fBWGPYRs%2Bilv6%2BB0ZgyIiuPVSBegVTDpGKYytBZgrINr17faxSnSLICnNBqQRP5myQqmvLhU163xkcolcnOHSppYw2WQBgaNlFAStx9CV%2BvDPNcZ38ESWafMW0Loi0Nw3OnUlXY8pNMudX%2BeI06YIk7cUWC2GRC6tczZF4fp%2BIQSElMD0wTGVlZiDFF8q1LKIj1gH5K00DZzgiLt%2FxUCb6SaEmCYzEeNsWvf%2Fg5HOA%2BzvbYB%2BgLYWxmXSm0puXm9zcZ%2Fjw0DLkXqh4Gh7N94jngQP0LZzt%2FHwMG9%2B3xySB%2F%2BJglqu%2BnE7aXRydL5U%2FWm5pMKXO98r4Q1NWvfGEiPgK%2FQpa9%2FUMjDi35K0BjqGAo4vvliFOkt%2FCkEgsbdiCFWbEb79HHsx8kGYL4K%2B5CLWQ38FiCtMyFKd0Cx6xAV4nwoL1JrTNhfTMXEtnXsPYXNPS2vuSVkDGLC6n44cR9T0HB%2F8JZCrFvcQO2reeTn6PIUAYY%2F%2BNs3HtJ5SXNIBJp%2B5RWPQmv0BYDMwVkxWFGOr2%2FIRMwycdl34UH2mWvh1KN2n1V2BcV5ZkDwmIxVazUH88buQErhuT9e7gRa0LstXDIvnaRhB9HCDCPkgRKw%2FSEHCFVnIrB8c7Ltarw%2BGGYsTIUs%2Bysoc5dAUpzkZ7hjELv0dIJdZfeM7GGVDt1i8KmV0LTHrk8OJiASNpq5IJlM9Lw0NnAg%3D&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Date=20240703T063433Z&X-Amz-SignedHeaders=host&X-Amz-Expires=43200&X-Amz-Credential=ASIA3VELI2DCTDFO2THD%2F20240703%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Signature=37ad9507a1b46bbd287677792cfd12e0479cfd808768b8adc964a680a78c43b3" %}



4-4 summarize



```python
import json
import boto3
import datetime
import os
from botocore.config import Config

my_config_west = Config(
    region_name = 'us-west-2',
    retries = {
        'max_attempts': 10,
        'mode': 'standard'
        }
    )
my_config_east = Config(
    region_name = 'us-east-1',
    retries = {
        'max_attempts': 10,
        'mode': 'standard'
        }
    )


transcribe = boto3.client("transcribe", config = my_config_west)
s3 = boto3.resource('s3')
mediaconvert = boto3.client('mediaconvert', config = my_config_west)

retry_config = Config(
    region_name='us-west-2',
    retries={
        "max_attempts": 10,
        "mode": "standard",
    },
    read_timeout=1200,
)

# modelId = "anthropic.claude-instant-v1"  # (Change this to try different model versions)
modelId = "anthropic.claude-3-sonnet-20240229-v1:0"
accept = "application/json"
contentType = "application/json"

bedrock_runtime = boto3.client(service_name='bedrock-runtime', config=retry_config)

def bedrock_streamer(response):
    stream = response.get('body')
    answer = ""
    i = 1
    if stream:
        for event in stream:
            chunk = event.get('chunk')
            if chunk:
                chunk_obj = json.loads(chunk.get('bytes').decode())
                if "delta" in chunk_obj:
                    delta = chunk_obj['delta']
                    if "text" in delta:
                        text = delta['text']
                        print(text, end="")
                        answer += str(text)
                        i += 1
    return answer

def set_job(bucket, input_key, output_key, start_time, end_time):
    job_settings = {
        'Settings': {
            "Inputs": [
                {
                    "TimecodeSource": "ZEROBASED",
                    "VideoSelector": {},
                    "AudioSelectors": {
                        "Audio Selector 1": {
                            "DefaultSelection": "DEFAULT"
                        }
                    },
                    "FileInput": f's3://{bucket}/{input_key}',
                    "InputClippings": [
                        {
                            "StartTimecode": start_time,
                            "EndTimecode": end_time
                        }
                    ]
                }
            ],
            'OutputGroups': [{
                'CustomName': 'Output Group',
                'OutputGroupSettings': {
                    'Type': 'FILE_GROUP_SETTINGS',
                    'FileGroupSettings': {
                        'Destination': f's3://{bucket}/{output_key}'
                    }
                },
                "Outputs": [
                  {
                    "ContainerSettings": {
                      "Container": "MP4",
                      "Mp4Settings": {}
                    },
                    "VideoDescription": {
                      "CodecSettings": {
                        "Codec": "H_264",
                        "H264Settings": {
                          "RateControlMode": "QVBR",
                          "SceneChangeDetect": "TRANSITION_DETECTION",
                          "MaxBitrate": 5000000
                        }
                      }
                    },
                    "AudioDescriptions": [
                        {
                            "AudioSourceName": "Audio Selector 1",
                            "CodecSettings": {
                                "Codec": "AAC",
                                "AacSettings": {
                                    "Bitrate": 96000,
                                    "CodingMode": "CODING_MODE_2_0",
                                    "SampleRate": 48000
                                }
                            }
                        }
                    ]
                  }
                ],
                "CustomName": "weverse"
              }
            ],
            "TimecodeConfig": {
              "Source": "ZEROBASED"
            },
            "FollowSource": 1
          },
          "Role": "arn:aws:iam::801303482565:role/mediaconvert-summarize-role",
          "StatusUpdateInterval": "SECONDS_60",
          "BillingTagsSource": "QUEUE",
          "Queue": "arn:aws:mediaconvert:us-east-1:801303482565:queues/Default"
        }

    return job_settings

def merge_transcribe(transcribe_result):
    new_transcribe = []
    data = {'start_time': 0, 'end_time': 0, 'content': '', 'scene': ''}
    for transcribe in transcribe_result['items']:
        #print(transcribe)
        if transcribe['type'] != 'punctuation':
            data['content'] += transcribe['alternatives'][0]['content'] + ' '
            if data['start_time'] == 0:
                data['start_time'] = float(transcribe['start_time'])
            end_time = float(transcribe['end_time'])
        else:
            data['end_time'] = end_time
            new_transcribe.append(data)
            data = {'start_time': 0, 'end_time': 0, 'content': '', 'scene': ''}
    return new_transcribe

def check_transcribe_job(key):
    
    transcribe_job_name = 'job_' + key.split('/')[-1].split('.')[0]
    
    max_tries = 60
    while max_tries > 0:
        max_tries -= 1
        job = transcribe.get_transcription_job(TranscriptionJobName=transcribe_job_name)
        job_status = job["TranscriptionJob"]["TranscriptionJobStatus"]
        if job_status in ["COMPLETED", "FAILED"]:
            print(f"Job {transcribe_job_name} is {job_status}.")
            if job_status == "COMPLETED":
                print(
                    f"Download the transcript from\n"
                    f"\t{job['TranscriptionJob']['Transcript']['TranscriptFileUri']}."
                )
            break
        else:
            print(f"Waiting for {transcribe_job_name}. Current status is {job_status}.")
        time.sleep(10)

def blip_with_merge(blip_results, new_transcribe, SAMPLE_EVERY_SEC=2):
            
    for i, result in enumerate(blip_results):
        for transcribe in new_transcribe:
            if (transcribe['start_time'] <= ((2*i+1) * SAMPLE_EVERY_SEC)/2) \
                    and (transcribe['end_time'] >= ((2*i + 1) * SAMPLE_EVERY_SEC)/2):
                print('check', transcribe['start_time'], transcribe['end_time'])    
                if result not in transcribe['scene']:
                    transcribe['scene'] += result + ', '
                break

        print(i * SAMPLE_EVERY_SEC, result)
    return new_transcribe
    
def call_claude_sonet(prompt, base64_string=None, streaming=True, temperature=0, top_p=.999):
    if base64_string is None:
        prompt_config = {
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 4096,
            "temperature": temperature,
            "top_k": 350,
            "top_p": top_p,
            "messages": [
                {
                    "role": "user",
                    "content": [
                        {"type": "text", "text": prompt},
                    ],
                }
            ],
        }
    else:
        prompt_config = {
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 4096,
            "temperature": 0,
            "top_k": 350,
            "top_p": top_p,
            "messages": [
                {
                    "role": "user",
                    "content": [
                        {
                            "type": "image",
                            "source": {
                                "type": "base64",
                                "media_type": "image/png",
                                "data": base64_string,
                            },
                        },
                        {"type": "text", "text": prompt},
                    ],
                }
            ],
        }

    body = json.dumps(prompt_config)

    modelId = "anthropic.claude-3-sonnet-20240229-v1:0"
    accept = "application/json"
    contentType = "application/json"

    if streaming:

        response = bedrock_runtime.invoke_model_with_response_stream(
            body=body, modelId=modelId, accept=accept, contentType=contentType
        )
        results = bedrock_streamer(response)
    else:
        response = bedrock_runtime.invoke_model(
            body=body, modelId=modelId, accept=accept, contentType=contentType
        )
        response_body = json.loads(response.get("body").read())
        results = response_body.get("content")[0].get("text")
    return results
    
def modifyFormat(result):
    x = result.splitlines(keepends=False)
    final_results = []
    for line in x:
        if line == '':
            continue
        text = line.split(',')
        print(text)
        if len(text) > 3:
            text = [text[0], text[1], ','.join(text[2:])]

        data = {}
        data['start_time'] = text[0].split('-')[1] if 'start_time' in text[0] else text[0]
        data['end_time'] = text[1].split('-')[1] if 'end_time' in text[1] else text[1]
        data['title'] = text[2].split('-')[1] if 'title' in text[2] else text[2]
        data['start_time'] = int(data['start_time'].split(':')[0]) if ':' in data['start_time'] else int(float(data['start_time']))
        if data['end_time'] == '끝':
            data['end_time'] = video_len
            data['end_time'] = int(data['end_time'])
        else:
            data['end_time'] = int(data['end_time'].split(':')[0]) if ':' in data['end_time'] else int(float(data['end_time']))
        data['start_time'] = (lambda x : '0' + x if len(x) < 8 else x)(str(datetime.timedelta(seconds = data['start_time'])))
        data['end_time'] = (lambda x : '0' + x if len(x) < 8 else x)(str(datetime.timedelta(seconds = data['end_time'])))
        data['start_time'] = data['start_time'] + ':00'
        data['end_time'] = data['end_time'] + ':00'

        final_results.append(data)
    print('\nfinal_result\n')
    print(final_results)
    return final_results

def lambda_handler(event, context):
    # TODO implement
    print(event['responsePayload'])
    bucket = event['responsePayload']['bucket']
    key = event['responsePayload']['key']
    blip_results = event['responsePayload']['body']
    interval = event['responsePayload']['interval']
    
    json_file = 'transcribe/'+key.split('/')[-1].split('.')[0]+'.json'
    check_transcribe_job(key)
    
    s3.Bucket(bucket).download_file(json_file, '/tmp/transcribe.json')
    
    transcribe_json = None
    with open('/tmp/transcribe.json') as f:
        transcribe_json = json.load(f)
        print('transcribe result', transcribe_json)
        
    if transcribe_json is not None:
        transcribe_result = merge_transcribe(transcribe_json)
        
        transcribe_result = blip_with_merge(blip_results, transcribe_result, interval)
    
    print(transcribe_result)
    
    prompt = """
    당신은 video analyst입니다. 아래의 <example>안에는 start_time, end_time, content, scene으로 구성된 자막이 있습니다. <condition></condition>에 해당하는 결과를 출력합니다. please think about the question within <condition></condition> XML tags.
    """

    example = """
    <example>
    {}
    </example>
    """.format(transcribe_result)
    
    condition = """
    <condition>
    대화의 주제에 대한 타임스탬프를 나열하고 내용을 요약해주세요. 타임스탬프는 오직 초단위만 가능합니다. 대화의 주제는 상세하게 나열해야 하며 시작과 끝이 자연스러워야 하고 빈 구간이 없어야합니다.
    </condition>
    """
    
    prompt = prompt + "\n" + example + "\n" + condition

    result = call_claude_sonet(prompt, streaming=False, temperature=1)
    print(result)
    
    condition = """
    최종 출력 결과는 오직 'start_time,end_time,title' 포맷으로 출력합니다. start_time, end_time은 반드시 초단위로 출력합니다. 유사한 주제를 하나의 주제로 통합해주세요.
    """
    prompt = result + "\n\n" + condition
    print('\n')
    print('\nafter')
    result = call_claude_sonet(prompt, temperature=0, top_p=0)
    
    print(result)
    
    final_results = modifyFormat(result)
    
    for final_result in final_results:
        print(final_result)
        result_path = key.split('/')[-1].split('.')[0]
        job_settings = set_job(bucket, input_key=key, output_key='result/'+result_path+'/'+final_result['title'], start_time=final_result['start_time'], end_time=final_result['end_time'])

        try:
            response = mediaconvert.create_job(**job_settings)
            job_id = response['Job']['Id']
            print(f'MediaConvert 작업이 제출되었습니다. 작업 ID: {job_id}')
        except Exception as e:
            print(f'오류 발생: {e}')
    
    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }

```

{% embed url="https://enm-share.s3.us-west-2.amazonaws.com/summarizeVideo-adb5b360-6958-430a-801d-0b8c9eaa1446.zip?response-content-disposition=inline&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEMz%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLWVhc3QtMSJIMEYCIQCxnocurDQij%2FGhhHdnLiKPoOXfXayfEceBuj7mWs6nBAIhAKhQIlMXtBIgnbvXpIPo%2FUvTD594Hv0iTIpRvf1pkBdTKukCCIX%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEQABoMODAxMzAzNDgyNTY1IgwqSKi951rtq6VaigsqvQIwsSdJB10B7xk9O22hXRqsdxWu%2BRcnecHyWdgqkcOtZvVs7p5NKhLBGFqRMGGSO4w2h7X6Xt3fBWGPYRs%2Bilv6%2BB0ZgyIiuPVSBegVTDpGKYytBZgrINr17faxSnSLICnNBqQRP5myQqmvLhU163xkcolcnOHSppYw2WQBgaNlFAStx9CV%2BvDPNcZ38ESWafMW0Loi0Nw3OnUlXY8pNMudX%2BeI06YIk7cUWC2GRC6tczZF4fp%2BIQSElMD0wTGVlZiDFF8q1LKIj1gH5K00DZzgiLt%2FxUCb6SaEmCYzEeNsWvf%2Fg5HOA%2BzvbYB%2BgLYWxmXSm0puXm9zcZ%2Fjw0DLkXqh4Gh7N94jngQP0LZzt%2FHwMG9%2B3xySB%2F%2BJglqu%2BnE7aXRydL5U%2FWm5pMKXO98r4Q1NWvfGEiPgK%2FQpa9%2FUMjDi35K0BjqGAo4vvliFOkt%2FCkEgsbdiCFWbEb79HHsx8kGYL4K%2B5CLWQ38FiCtMyFKd0Cx6xAV4nwoL1JrTNhfTMXEtnXsPYXNPS2vuSVkDGLC6n44cR9T0HB%2F8JZCrFvcQO2reeTn6PIUAYY%2F%2BNs3HtJ5SXNIBJp%2B5RWPQmv0BYDMwVkxWFGOr2%2FIRMwycdl34UH2mWvh1KN2n1V2BcV5ZkDwmIxVazUH88buQErhuT9e7gRa0LstXDIvnaRhB9HCDCPkgRKw%2FSEHCFVnIrB8c7Ltarw%2BGGYsTIUs%2Bysoc5dAUpzkZ7hjELv0dIJdZfeM7GGVDt1i8KmV0LTHrk8OJiASNpq5IJlM9Lw0NnAg%3D&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Date=20240703T063453Z&X-Amz-SignedHeaders=host&X-Amz-Expires=43200&X-Amz-Credential=ASIA3VELI2DCTDFO2THD%2F20240703%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Signature=29470717fa13386ef295d370cf0eb6d0169b70aea6b3fe0905143013630dd450" %}
