StrukCerdas: Smart OCR Bill Splitter

This project is a complete smart bill splitting system. Instead of manually inputting receipt items one by one, this application allows users to scan a receipt, and the system will automatically extract all items using OCR.

Afterward, users can assign who ordered which item, and the backend will intelligently calculate the most efficient way to settle the bill with the minimum number of transactions.

🎯 Project Goal

The main goal is to eliminate the hassle of transferring money back and forth after a group meal. The application will calculate the "who-owes-who" at the end.

Example:

Andi owes Rp 50,000 to Budi.

Budi owes Rp 50,000 to Cici.

The application will not suggest two separate transfers.

It will optimize this into: Andi pays Rp 50,000 to Cici.

<!--
[REPLACE WITH YOUR ANDROID APP SCREENSHOT]
-->

🚀 Key Features

Android App (Kotlin): A mobile interface to capture the receipt photo.

OCR Engine (Pytesseract): Extracts all items, prices, taxes, discounts, and service fees from the receipt image.

Assignment Interface (Android): (App Feature) Allows users to "claim" which items they ordered or shared.

Smart Calculation API (FastAPI): An API endpoint that receives the assignment data and calculates the total debt per person.

Debt Optimization Algorithm: Calculates the minimum number of transactions required to settle all debts.

🛠️ Tech Stack

Frontend (Mobile App)

Platform: Android

Language: Kotlin

Tasks: (1) Send image to /scan, (2) Display OCR results, (3) Let users assign items to people, (4) Send assignment data to /calculate-split, (5) Display the final optimized transactions.

Backend (API & ML Model)

API Framework: FastAPI (Modern, fast, asynchronous).

OCR Engine: Pytesseract.

Logic: Pure Python business logic for calculating and optimizing debts.

Architecture Diagram

This architecture has two main flows:

Flow 1: Scanning Scenario

[Android App]        [Backend API (FastAPI)]          [Pytesseract]
---------------      -----------------------          -------------
| (1) Take Photo|                                       |
---------------      -----------------------          -------------
       |                                              |
       |---(2) POST /scan [Image File]--------------->| (3) Process OCR |
       |                                              |                 |
       |                                              | <---[Raw Text]----|
       |                                              | (4) Extract Items|
<------(5) JSON [Raw Item List]-----------------------|                 |
---------------      -----------------------          -------------
| (6) Display   |
| items & start |
| "Assigning"   |
---------------


Flow 2: Calculation Scenario (Bill Splitting)

[Android App]                     [Backend API (FastAPI)]
---------------                   -----------------------
| (7) User finishes|
|     assigning items|
|     to people     |
---------------                   -----------------------
       |                                            |
       |---(8) POST /calculate-split [JSON Data]--->| (9) Calculate total debt |
       |     (Who ate what, who paid)               |     per person.          |
       |                                            | (10) Run debt           |
       |                                            |      optimization algo.  |
       |                                            |                          |
<------(11) JSON [Minimal Transaction List]---------|
---------------                   -----------------------
| (12) Display:  |
| "Budi pays     |
|  Andi Rp 50.000"|
---------------


⚙️ Backend Setup & Installation

To run the backend server on your local machine.

1. Prerequisites

You MUST install the Tesseract OCR Engine on your system. pytesseract is just a Python wrapper.

Windows: Download the installer from Tesseract at UB Mannheim. Make sure to add it to your system PATH during installation.

macOS: brew install tesseract

Linux (Ubuntu): sudo apt install tesseract-ocr

2. Clone & Setup Environment

# (Assuming you create a 'backend' folder for these files)
cd backend

# 1. Create a virtual environment
python -m venv venv

# 2. Activate the virtual environment
# Windows
.\venv\Scripts\activate
# macOS/Linux
source venv/bin/activate

# 3. Install all dependencies
pip install -r requirements.txt


3. Run the Server

From inside the backend/ folder, run the Uvicorn server:

# The server will run on [http://127.0.0.1:8000](http://127.0.0.1:8000)
uvicorn main:app --reload


Your server is now live. You can open http://12_7.0.0.1:8000/docs in your browser to see the automatic API documentation (Swagger UI) and even test the endpoints directly from there.

📱 Android App Setup (Frontend)

Open the Android project in Android Studio.

Find the file where you define API constants (e.g., ApiService.kt or Constants.kt).

Change the BASE_URL to point to your computer's local IP address (not 127.0.0.1 or localhost, as the emulator/phone won't recognize it).

Find your IP address (e.g., on Windows ipconfig, on macOS/Linux ifconfig). Example: 192.168.1.10.

Change the URL to: const val BASE_URL = "http://192.168.1.10:8000/"

Ensure your phone/emulator is on the same WiFi network as your backend computer.

Build and run the app.

📚 API Documentation

The main endpoints provided by the backend:

POST /scan

Purpose: To extract a raw list of items and costs from a receipt image.

Request Body: form-data with a key file (the image file).

Success Response (200):

{
  "filename": "receipt.jpg",
  "raw_text": "...",
  "details": {
    "items": [
      { "qty": 2, "name": "INDOMIE GORENG", "price": "6.000" },
      { "qty": 1, "name": "TEH KOTAK", "price": "3.500" }
    ],
    "total": "10.500",
    "discount": "0",
    "service_fee": "1.000"
  }
}


POST /calculate-split

Purpose: To calculate the bill split and optimize the transactions.

Request Body (JSON): This is the data sent from the Android app after users have assigned items.

{
  "people": ["Andi", "Budi", "Cici"],
  "items": [
    { "name": "Nasi Goreng", "price": 20000, "shared_by": ["Andi"] },
    { "name": "Ayam Bakar", "price": 25000, "shared_by": ["Budi"] },
    { "name": "Kentang Goreng", "price": 15000, "shared_by": ["Andi", "Budi", "Cici"] }
  ],
  "shared_fees": [
    { "name": "Pajak 10%", "amount": 6000 },
    { "name": "Service 5%", "amount": 3000 }
  ],
  "shared_discounts": [
    { "name": "Promo OVO", "amount": 5000 }
  ],
  "payers": [
    { "name": "Andi", "amount_paid": 64000 }
  ]
}


Success Response (200):

totals_per_person: A breakdown of the total amount each person should have paid.

transactions: The minimal list of transactions required to settle all debts.

{
  "totals_per_person": {
    "Andi": 26666.67,
    "Budi": 31666.67,
    "Cici": 5666.67
  },
  "transactions": [
    { "from_person": "Budi", "to_person": "Andi", "amount": 31666.67 },
    { "from_person": "Cici", "to_person": "Andi", "amount": 5666.67 }
  ]
}


🧠 How It Works (Extraction & Calculation)

Extraction (Regex): The extract_receipt_details function (called by /scan) uses Regular Expressions (Regex) to find specific patterns ([Quantity] [Item Name] [Price]) in the raw OCR text.

Calculation (Debt Optimization): The calculate_minimum_transactions function (called by /calculate-split) is the core of the split-bill logic.

It calculates each person's net balance (Total Paid - Total Owed).

It separates people into "Debtors" (negative balance) and "Creditors" (positive balance).

It then matches the largest debtor with the largest creditor to generate the minimum number of transactions.