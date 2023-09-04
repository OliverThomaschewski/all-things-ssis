# ReadMe

This Script can be used to access data from google sheets and save the results in a csv.

## Dependencies

pip install --upgrade google-api-python-client google-auth-httplib2 google-auth-oauthlib 

pip install pandas

## Prerequisites

API Service hast du be set up in Google Console and credentials.json must be downloaded

## Usage

- Fill in the IDs of the Spreadheet
- Paths to credentials.json and filename and path where you want the result.csv
- Hope for the best


If this is going to be used in a SSIS Execution Process Task, the file has to be converted to an .exe (i used pyinstaller --onefile --noconsole filename.py)




SSIS Setup

<img width="743" alt="image" src="https://github.com/OliverThomaschewski/googlesheets_ssis_script/assets/95187084/5807d999-a32b-4604-aa1a-9da4cc754b0c">
