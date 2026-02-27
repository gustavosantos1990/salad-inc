# GitLab Welcome Email Configuration

This guide configures GitLab to send welcome emails to new users when they first log in via Keycloak SSO.

## Prerequisites

- GitLab installed and running
- Mail server (mail.salad.local) configured and accessible
- Keycloak SSO integration working

---

## Step 1: Configure GitLab SMTP Settings

### 1.1. SSH into GitLab VM

```bash
ssh root@repository.salad.local
```

### 1.2. Edit GitLab Configuration

```bash
vi /etc/gitlab/gitlab.rb
```

### 1.3. Add/Update Email Configuration

Find the email settings section (or add it) and configure:

```ruby
################################################################################
## Email Settings
################################################################################

# Enable email notifications
gitlab_rails['gitlab_email_enabled'] = true
gitlab_rails['gitlab_email_from'] = 'gitlab@salad.local'
gitlab_rails['gitlab_email_display_name'] = 'GitLab - Salad Inc'
gitlab_rails['gitlab_email_reply_to'] = 'noreply@salad.local'
gitlab_rails['gitlab_email_subject_suffix'] = ''

################################################################################
## SMTP Settings
################################################################################

gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "mail.salad.local"
gitlab_rails['smtp_port'] = 25
gitlab_rails['smtp_domain'] = "salad.local"
gitlab_rails['smtp_authentication'] = false
gitlab_rails['smtp_enable_starttls_auto'] = false
gitlab_rails['smtp_tls'] = false
gitlab_rails['smtp_openssl_verify_mode'] = 'none'

# Optional: Set timeout values
gitlab_rails['smtp_pool'] = false
```

**Save and exit** (`:wq` in vi)

---

## Step 2: Reconfigure GitLab

Apply the configuration changes:

```bash
gitlab-ctl reconfigure
```

This will take a few minutes. Wait for it to complete.

---

## Step 3: Test Email Configuration

### 3.1. Open GitLab Rails Console

```bash
gitlab-rails console
```

Wait for the console to load (may take 30-60 seconds).

### 3.2. Send Test Email

In the Rails console, run:

```ruby
Notify.test_email('alice.dev@salad.local', 'GitLab Email Test', 'This is a test email from GitLab.').deliver_now
```

You should see output like:
```
Delivered mail ... (XXXms)
=> #<Mail::Message:...>
```

### 3.3. Exit Console

```ruby
exit
```

### 3.4. Verify Email Delivery

On the mail server VM, check if the email was delivered:

```bash
ssh root@mail.salad.local
ls -la /var/mail/vhosts/salad.local/alice.dev/Maildir/new/
```

You should see a new email file.

Or check Postfix logs:

```bash
tail -30 /var/log/messages | grep postfix
```

---

## Step 4: Enable Welcome Emails for New Users

### 4.1. Configure User Settings

Edit `/etc/gitlab/gitlab.rb` again:

```bash
vi /etc/gitlab/gitlab.rb
```

Add these settings:

```ruby
################################################################################
## User Settings
################################################################################

# Send notification email when new users are created
gitlab_rails['gitlab_default_can_create_group'] = true
gitlab_rails['gitlab_username_changing_enabled'] = false

# Disable signup (we use SSO)
gitlab_rails['gitlab_signup_enabled'] = false

# Send emails to new users
gitlab_rails['omniauth_block_auto_created_users'] = false
gitlab_rails['omniauth_auto_link_user'] = ['openid_connect']
```

**Save and exit**

### 4.2. Reconfigure Again

```bash
gitlab-ctl reconfigure
```

---

## Step 5: Customize Welcome Email Template (Optional)

### 5.1. Locate Email Templates

GitLab email templates are in:

```bash
cd /opt/gitlab/embedded/service/gitlab-rails/app/views/notify
ls -la
```

### 5.2. Create Custom Template Directory

```bash
mkdir -p /opt/gitlab/embedded/service/gitlab-rails/ee/app/views/notify
```

### 5.3. Copy and Customize Template

Copy the new user email template:

```bash
cp /opt/gitlab/embedded/service/gitlab-rails/app/views/notify/new_user_email.html.haml \
   /opt/gitlab/embedded/service/gitlab-rails/ee/app/views/notify/new_user_email.html.haml
```

Edit the custom template:

```bash
vi /opt/gitlab/embedded/service/gitlab-rails/ee/app/views/notify/new_user_email.html.haml
```

Example customization:

```haml
%p
  Hi #{@user.name}!

%p
  Welcome to GitLab at Salad Inc!

%p
  Your GitLab account has been successfully created when you logged in via Keycloak SSO.

%p
  %strong Your account details:
%ul
  %li Username: #{@user.username}
  %li Email: #{@user.email}

%p
  %strong Getting Started:
%ul
  %li Access GitLab: #{link_to gitlab_url, gitlab_url}
  %li Explore your projects and repositories
  %li Join your team's group to collaborate

%p
  Need help? Contact your team administrator.

%p
  Best regards,
  %br
  The Salad Inc DevOps Team
```

**Note:** After GitLab upgrades, you may need to recreate custom templates.

---

## Step 6: Configure GitLab Admin Settings (Web UI)

### 6.1. Access GitLab Admin Area

1. Log in to GitLab as admin: `https://gitlab.salad.local`
2. Click **Admin Area** (wrench icon) in the top menu

### 6.2. Configure Email Settings

1. Go to **Settings** → **Preferences**
2. Expand **Email** section
3. Configure:
   - ✅ **Enable email notifications**
   - **Email notification email**: `gitlab@salad.local`
   - **Custom hostname**: `gitlab.salad.local`
4. Click **Save changes**

### 6.3. Configure Sign-up Restrictions

1. Go to **Settings** → **General**
2. Expand **Sign-up restrictions**
3. Configure:
   - ❌ **Sign-up enabled** (unchecked - we use SSO)
   - ✅ **Send confirmation email on sign-up** (checked)
   - ✅ **Require admin approval for new sign-ups** (optional)
4. Click **Save changes**

---

## Step 7: Test Welcome Email with New User

### 7.1. Create Test Scenario

Since we use SSO, the welcome email is triggered when:
1. A user logs in via Keycloak for the first time
2. GitLab auto-creates their account
3. GitLab sends the welcome email

### 7.2. Test with New User

1. Log out of GitLab
2. Log in as a user who hasn't logged into GitLab before (e.g., `charlie.dev`)
3. Complete the Keycloak SSO flow
4. GitLab should auto-create the account

### 7.3. Verify Email Was Sent

Check GitLab logs:

```bash
gitlab-rails console
```

```ruby
# Check recent emails sent
ActionMailer::Base.deliveries.last
exit
```

Or check mail server logs:

```bash
ssh root@mail.salad.local
tail -50 /var/log/messages | grep -i "charlie.dev"
```

Check mailbox:

```bash
ls -la /var/mail/vhosts/salad.local/charlie.dev/Maildir/new/
```

---

## Step 8: Monitor Email Delivery

### 8.1. Check GitLab Email Queue

```bash
gitlab-rails console
```

```ruby
# Check email queue status
Sidekiq::Queue.new('mailers').size

# View recent email jobs
Sidekiq::Queue.new('mailers').each { |job| puts job.args }

exit
```

### 8.2. Check GitLab Logs

```bash
tail -f /var/log/gitlab/gitlab-rails/production.log | grep -i mail
```

### 8.3. Check Mail Server Logs

```bash
ssh root@mail.salad.local
tail -f /var/log/messages | grep postfix
```

---

## Troubleshooting

### Issue: Emails Not Sending

**Check SMTP connection:**

```bash
gitlab-rails console
```

```ruby
ActionMailer::Base.smtp_settings
# Should show your mail.salad.local configuration

# Test connection
require 'net/smtp'
Net::SMTP.start('mail.salad.local', 25) do |smtp|
  puts "Connected successfully!"
end

exit
```

**Check GitLab email queue:**

```bash
gitlab-rails console
```

```ruby
Sidekiq::Queue.new('mailers').size
# If > 0, emails are queued but not sending
exit
```

**Restart Sidekiq:**

```bash
gitlab-ctl restart sidekiq
```

### Issue: Welcome Email Not Triggered

**Verify auto-creation is enabled:**

```bash
gitlab-rails console
```

```ruby
ApplicationSetting.current.omniauth_block_auto_created_users
# Should be false

exit
```

**Check if user was auto-created:**

```bash
gitlab-rails console
```

```ruby
User.find_by(username: 'charlie.dev')
# Should show user details

exit
```

### Issue: Email Delivered but Not Received

**Check mail server:**

```bash
ssh root@mail.salad.local

# Check if email was received
tail -100 /var/log/messages | grep "charlie.dev"

# Check mailbox
ls -la /var/mail/vhosts/salad.local/charlie.dev/Maildir/new/
```

**Check Dovecot logs:**

```bash
tail -100 /var/log/messages | grep dovecot
```

---

## Verification Checklist

- [ ] GitLab SMTP settings configured in `gitlab.rb`
- [ ] `gitlab-ctl reconfigure` completed successfully
- [ ] Test email sent successfully via Rails console
- [ ] Test email received in user's mailbox
- [ ] Welcome email settings enabled
- [ ] New user logs in via Keycloak SSO
- [ ] GitLab auto-creates user account
- [ ] Welcome email sent to new user
- [ ] Email appears in user's mailbox
- [ ] Email content is correct and readable

---

## Configuration Summary

### GitLab Configuration (`/etc/gitlab/gitlab.rb`)

```ruby
# Email Settings
gitlab_rails['gitlab_email_enabled'] = true
gitlab_rails['gitlab_email_from'] = 'gitlab@salad.local'
gitlab_rails['gitlab_email_display_name'] = 'GitLab - Salad Inc'
gitlab_rails['gitlab_email_reply_to'] = 'noreply@salad.local'

# SMTP Settings
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "mail.salad.local"
gitlab_rails['smtp_port'] = 25
gitlab_rails['smtp_domain'] = "salad.local"
gitlab_rails['smtp_authentication'] = false
gitlab_rails['smtp_enable_starttls_auto'] = false
gitlab_rails['smtp_tls'] = false
gitlab_rails['smtp_openssl_verify_mode'] = 'none'

# User Settings
gitlab_rails['gitlab_signup_enabled'] = false
gitlab_rails['omniauth_block_auto_created_users'] = false
gitlab_rails['omniauth_auto_link_user'] = ['openid_connect']
```

### Required Commands

```bash
# Apply configuration
gitlab-ctl reconfigure

# Test email
gitlab-rails console
Notify.test_email('user@salad.local', 'Test', 'Body').deliver_now
exit

# Check logs
tail -f /var/log/gitlab/gitlab-rails/production.log
```

---

## Next Steps

After welcome emails are working:

1. Customize email templates for branding
2. Set up email notifications for other events (merge requests, issues, etc.)
3. Configure email reply-by-email (optional)
4. Set up email forwarding rules (optional)

---

## Additional Resources

- GitLab SMTP Documentation: https://docs.gitlab.com/omnibus/settings/smtp.html
- Email Templates: `/opt/gitlab/embedded/service/gitlab-rails/app/views/notify/`
- GitLab Logs: `/var/log/gitlab/gitlab-rails/production.log`
