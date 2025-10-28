# Embedded-iot-project
Developed a real-time solar dehydration monitoring system using a specialized camera (PSNR: 27 29 dB), replacing manual inspections to enhance efficiency, transparency, and sustainability.



MAIN PROGRAM TO CAPTURE IMAGE TOSTORE IN A FOLDER UPLOAD IT TO GOOGLE DRIVE & CALCULATE PSNR VALUES
import time
import requests  # For sending HTTP requests
import csv  # For saving data to CSV file
import os
from gpiozero import LED
from datetime import datetime
import numpy as np
import cv2
from google.oauth2 import service_account
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload
import DS18B20
import DHT11

# GPIO Pin Configuration
HUMID_PIN = 17  
LIGHT_PIN = 2  
#DS18B20=4
#DHT11=20

# ThingSpeak Configuration
THING_SPEAK_API_KEY = '9RRYE408KCY83WUY'
THINGSPEAK_URL = 'https://api.thingspeak.com/update?'

# Google Drive Configuration
SERVICE_ACCOUNT_FILE = '/home/pi/project/evident-syntax-444913-a5-39250395746d.json'
FOLDER_ID = '1vPDTEbKkqFvb32-u6v5TalhPMn-EWwJ7'
IMAGE_DIRECTORY = '/home/pi/New'
REFERENCE_IMAGE_PATH = '/home/pi/test2.jpg'

# User-Defined Thresholds
MIN_HUMIDITY = int(input("Enter the minimum humidity threshold: "))
MIN_TEMP = int(input("Enter the minimum temperature threshold: "))

# GPIO Setup using gpiozero
humidifier = LED(HUMID_PIN)
light = LED(LIGHT_PIN)

# Initialize CSV File
csv_file = "sensor_data.csv"
file_exists = os.path.exists(csv_file)

with open(csv_file, mode='a', newline='') as file:
    writer = csv.writer(file)
    if not file_exists:
        writer.writerow(["Timestamp", "Temperature", "Humidity", "Weight", "PSNR"])
# Function: Control Humidifier
def control_humidifier(humidity):
    if humidity < MIN_HUMIDITY:
        print("Humidity too low. Turning on the humidifier.")
        humidifier.on()
    else:
        print("Humidity sufficient. Turning off the humidifier.")
        humidifier.off()
# Function: Control Light
def control_light(temperature):
  
  if temperature < MIN_TEMP:
        print("Temperature too low. Turning on the light.")
        light.on()
    else:
        print("Temperature sufficient. Turning off the light.")
        light.off()
# Function: Upload to ThingSpeak
def upload_to_thingspeak(temperature, humidity, weight, psnr_value):
    payload = {
        'api_key': THING_SPEAK_API_KEY,
        'field1': temperature,
        'field2': humidity,
        'field4': psnr_value
    }
    try:
        response = requests.post(THINGSPEAK_URL, data=payload)
        if response.status_code == 200:
            print("Data uploaded successfully to ThingSpeak.")
        else:
            print(f"Failed to upload data. Status: {response.status_code}")
    except Exception as e:
        print(f"Error: {e}")
# Function: Save Data to CSV
def save_to_csv(timestamp, temperature, humidity, psnr_value):
    with open(csv_file, mode='a', newline='') as file:
        writer = csv.writer(file)
        writer.writerow([timestamp, temperature, humidity, psnr_value])
# Function: Authenticate Google Drive
def authenticate_google_drive():
    SCOPES = ['https://www.googleapis.com/auth/drive.file']
  
credentials = service_account.Credentials.from_service_account_file(
        SERVICE_ACCOUNT_FILE, scopes=SCOPES)
    return build('drive', 'v3', credentials=credentials)
# Function: Upload Image to Google Drive
def upload_to_drive(drive_service, file_path):
    file_name = os.path.basename(file_path)
    media = MediaFileUpload(file_path, mimetype='image/jpeg')
    file_metadata = {'name': file_name, 'parents': [FOLDER_ID]}
    file = drive_service.files().create(body=file_metadata, media_body=media, fields='id').execute()
    print(f"Uploaded {file_name} to Google Drive. File ID: {file['id']}")
    return file['id']
# Function: Calculate PSNR
def calculate_psnr(image1, image2):
    mse = np.mean((image1 - image2) ** 2)
    if mse == 0:
        return float('inf')
    return 10 * np.log10((255.0 ** 2) / mse)


# Main Function
def main():
    drive_service = authenticate_google_drive()
    try:
        while True:
            # Read sensor data
            temperature = DS18B20.read_temperature()
            humidity = DHT11.get_sensor_humidity()

            print(f"Temperature: {temperature}Â°C, Humidity: {humidity}% g")

  # Control Devices
           control_humidifier(humidity)
          control_light(temperature)

            # Capture Image
            image_filename = f"captured_image_{datetime.now().strftime('%Y%m%d_%H%M%S')}.jpg"
            image_path = os.path.join(IMAGE_DIRECTORY, image_filename)

   
            if os.system(f"fswebcam -r 1280x720 --no-banner {image_path}") != 0:
                print("Error capturing image.")
                continue

            # Upload to Google Drive
            upload_to_drive(drive_service, image_path)

            # Upload and Compare Images
            captured_image = cv2.imread(image_path)
            reference_image = cv2.imread(REFERENCE_IMAGE_PATH)

            if captured_image is None or reference_image is None:
                print("Error: Could not load one or both images.")
                continue

            reference_image = cv2.resize(reference_image, (captured_image.shape[1], captured_image.shape[0]))
            psnr_value = calculate_psnr(reference_image, captured_image)
            print(f"PSNR: {psnr_value} dB")


 # Save Data
            timestamp = time.strftime("%Y-%m-%d %H:%M:%S")
            save_to_csv(timestamp, temperature, humidity,  psnr_value)

            # Upload Data to ThingSpeak
            upload_to_thingspeak(temperature, humidity, psnr_value)

            # Ensure compliance with ThingSpeak API rate limit
            time.sleep(30)

    except KeyboardInterrupt:
        print("Exiting program.")

# Run Main
if __name__ == "__main__":
    main()
