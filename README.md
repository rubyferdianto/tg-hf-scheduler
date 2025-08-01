# Telegram Bot on Google Cloud Run

This project deploys a Telegram bot to Google Cloud Run that provides QR codes for course attendance.

## Features

- 🤖 Telegram bot with interactive keyboard menu
- 📅 Scheduled daily notifications at 7 PM Singapore time
- 🔗 QR code links for different course modules
- ☁️ Runs on Google Cloud Run with webhook integration

## Prerequisites

1. **Google Cloud Account**: You need a Google Cloud Project with billing enabled
2. **Telegram Bot**: Create a bot using [@BotFather](https://t.me/botfather) and get your bot token
3. **Google Cloud CLI**: Install and configure the [gcloud CLI](https://cloud.google.com/sdk/docs/install)

## Quick Deployment

1. **Clone this repository**:
   ```bash
   git clone <your-repo-url>
   cd tg-hf-scheduler
   ```

2. **Update the deployment script**:
   Edit `deploy.sh` and update:
   - `PROJECT_ID`: Your Google Cloud Project ID
   - `TELEGRAM_BOT_TOKEN`: Your bot token from BotFather

3. **Run the deployment**:
   ```bash
   ./deploy.sh
   ```

4. **Test your bot**:
   Send `/start` to your bot on Telegram!

## Manual Deployment

If you prefer to deploy manually:

### 1. Set up Google Cloud

```bash
# Set your project
gcloud config set project YOUR_PROJECT_ID

# Enable APIs
gcloud services enable cloudbuild.googleapis.com
gcloud services enable run.googleapis.com
gcloud services enable secretmanager.googleapis.com
```

### 2. Create secrets

```bash
# Store your Telegram bot token
echo -n "YOUR_BOT_TOKEN" | gcloud secrets create telegram-bot-token --data-file=-
```

### 3. Deploy to Cloud Run

```bash
# Deploy the service
gcloud run deploy telegram-bot \
    --source . \
    --platform managed \
    --region us-central1 \
    --allow-unauthenticated \
    --port 8080 \
    --memory 512Mi \
    --cpu 1 \
    --max-instances 10 \
    --set-env-vars "PORT=8080" \
    --set-secrets "TELEGRAM_BOT_TOKEN=telegram-bot-token:latest"
```

### 4. Configure webhook

```bash
# Get your service URL
SERVICE_URL=$(gcloud run services describe telegram-bot --platform managed --region us-central1 --format 'value(status.url)')

# Update with webhook URL
gcloud run services update telegram-bot \
    --platform managed \
    --region us-central1 \
    --set-env-vars "WEBHOOK_URL=$SERVICE_URL"

# Set the webhook
curl -X POST "$SERVICE_URL/set_webhook"
```

## Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `TELEGRAM_BOT_TOKEN` | Your Telegram bot token | Yes |
| `WEBHOOK_URL` | Your Cloud Run service URL | Yes (auto-set) |
| `PORT` | Port for the web server | No (default: 8080) |

## Bot Commands

- `/start` - Shows the main menu with QR code options

## Module QR Codes

The bot provides QR codes for 6 different modules:
- Module 1: RA576069
- Module 2: RA576073  
- Module 3: RA576079
- Module 4: RA576096
- Module 5: RA576098
- Module 6: RA576102

## Scheduled Notifications

The bot sends daily reminders at 7:00 PM Singapore time to check in for NTU courses.

## Local Development

For local development, you can run the bot in polling mode:

```bash
# Install dependencies
pip install -r requirements.txt

# Set environment variable
export TELEGRAM_BOT_TOKEN="your_bot_token"

# Option 1: Run in polling mode using command line argument
python app.py --polling

# Option 2: Run in polling mode using environment variable
export WEBHOOK_MODE=false
python app.py
```

## Architecture

- **Flask App** (`app.py`): Unified application that handles both webhook and polling modes
- **TelegramBot** (`telegram_bot.py`): Bot logic with webhook support  
- **Scheduler** (`scheduler.py`): Background job for daily notifications
- **Docker**: Containerized for Cloud Run deployment
- **Gunicorn**: Production WSGI server

## Cost Optimization

Cloud Run pricing is based on:
- **CPU and Memory**: Only charged when processing requests
- **Requests**: $0.40 per million requests
- **Always-free tier**: 2 million requests per month

For a Telegram bot, costs are typically very low (often free within the always-free tier).

## Troubleshooting

### Notifications Not Working

1. **Check if the scheduler is running**:
   ```bash
   curl https://YOUR_SERVICE_URL/
   ```
   Look for `"scheduler_running": true` in the response.

2. **Test manual notification**:
   ```bash
   curl -X POST https://YOUR_SERVICE_URL/trigger_notification
   ```

3. **Keep service warm** (recommended):
   ```bash
   # Update setup_scheduler.sh with your PROJECT_ID first
   ./setup_scheduler.sh
   ```
   This creates a Cloud Scheduler job that pings your service every 5 minutes to prevent cold starts.

4. **Check Cloud Run logs**:
   ```bash
   gcloud run services logs read telegram-bot --region us-central1 --limit 50
   ```

### Common Issues

1. **Container cold starts**: Cloud Run shuts down idle containers, stopping the background scheduler.
   - **Solution**: Use the `setup_scheduler.sh` script to create a keep-warm job.

2. **Event loop errors**: `RuntimeError: Event loop is closed`
   - These are usually harmless and related to webhook cleanup.

3. **Bot not responding**: Check Cloud Run logs:
   ```bash
   gcloud run services logs read telegram-bot --region us-central1
   ```

4. **Webhook issues**: Verify webhook is set:
   ```bash
   curl -X GET "https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getWebhookInfo"
   ```

5. **Secret access**: Ensure the Cloud Run service has access to secrets:
   ```bash
   gcloud secrets describe telegram-bot-token
   ```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/` | GET | Health check with scheduler status |
| `/ping` | GET | Simple ping for keep-alive |
| `/webhook` | POST | Telegram webhook endpoint |
| `/set_webhook` | POST | Configure Telegram webhook |
| `/trigger_notification` | POST | Manual notification trigger (testing) |

## Support

For issues and questions, please check the logs and ensure all environment variables are properly set.
