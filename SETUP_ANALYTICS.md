# Setting Up Google Analytics

Your blog is now configured to use Google Analytics! Follow these steps to complete the setup.

## Step 1: Create a Google Analytics Account

1. Go to [Google Analytics](https://analytics.google.com/)
2. Sign in with your Google account
3. Click "Start measuring" or "Admin" (bottom left)
4. Click "Create Account"

## Step 2: Set Up a Property

1. **Account Name**: Enter a name (e.g., "Prasenjit Gupta's Blog")
2. Click "Next"
3. **Property Name**: Enter "pgupta1980.github.io"
4. **Reporting Time Zone**: Select your time zone
5. **Currency**: Select your currency
6. Click "Next"

## Step 3: Configure Data Stream

1. Select "Web" as the platform
2. **Website URL**: `https://pgupta1980.github.io`
3. **Stream Name**: "Blog"
4. Click "Create stream"

## Step 4: Get Your Measurement ID

After creating the stream, you'll see your **Measurement ID** (format: `G-XXXXXXXXXX`)

Copy this ID!

## Step 5: Add Measurement ID to Your Blog

1. Open `_config.yml` in your repository
2. Find this line:
   ```yaml
   google_analytics: G-XXXXXXXXXX
   ```
3. Replace `G-XXXXXXXXXX` with your actual Measurement ID
4. Commit and push:
   ```bash
   git add _config.yml
   git commit -m "Add Google Analytics tracking ID"
   git push
   ```

## Step 6: Verify Installation

1. Visit your blog: https://pgupta1980.github.io/
2. Open Google Analytics
3. Go to Reports → Realtime
4. You should see your visit appear within a few seconds!

## What You'll Track

Once set up, Google Analytics will automatically track:

- **Visitors**: Number of unique visitors
- **Page Views**: Which posts are most popular
- **Traffic Sources**: How people find your blog (Google, social media, direct)
- **Geography**: Where your visitors are from
- **Devices**: Desktop vs mobile vs tablet
- **Engagement**: Time on page, bounce rate
- **Real-time**: Live view of current visitors

## Privacy Considerations

Google Analytics respects user privacy:
- It's GDPR compliant
- Users can opt out via browser extensions
- Data is anonymized
- You can configure data retention settings

## Useful Reports

### Realtime Overview
See who's on your site right now

### Acquisition
How users find your blog (organic search, social media, direct)

### Engagement
Which pages and posts are most popular

### Demographics
Age, gender, interests of your audience (if enabled)

## Pro Tips

1. **Set up Goals**: Track specific actions (newsletter signups, clicks)
2. **Enable Demographics**: In Admin → Property → Data Settings
3. **Link to Search Console**: See which Google searches bring traffic
4. **Create Custom Reports**: Build dashboards for your needs
5. **Set up Alerts**: Get notified of traffic spikes or issues

## Troubleshooting

### Not seeing data?
- Wait 24-48 hours for initial data
- Check your Measurement ID is correct
- Make sure your site is published and live
- Try visiting your site in an incognito window

### Want to test immediately?
1. Visit your blog
2. Open Google Analytics → Reports → Realtime
3. You should see yourself as a visitor!

---

**Need help?** Check the [Google Analytics Help Center](https://support.google.com/analytics)
