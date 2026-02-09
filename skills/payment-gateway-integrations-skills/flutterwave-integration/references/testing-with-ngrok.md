## Testing Flutterwave Webhooks Locally with ngrok

1. Install ngrok: npm i -g ngrok
2. Start your server: npm run dev
3. Expose port: ngrok http 3000
4. Copy the https URL
5. Set webhook URL in Flutterwave dashboard to:
   https://xxxx.ngrok.io/api/flutterwave/webhook
6. Send test event from dashboard
