# BlueView Free Trial Implementation Guide

## 🎯 **How the Free Trial System Works**

Your website now captures trial signups via Netlify Forms. Here's how to connect it to your actual app:

---

## 📧 **1. Lead Processing Flow**

### **Current Setup:**
1. User fills out signup form → Netlify captures data
2. User sees success page → confirmation they're signed up
3. You get email notification → time to create their trial account

### **What You Need to Build:**

#### **A. Netlify Webhook Handler**
```javascript
// Example webhook endpoint (Node.js/Express)
app.post('/netlify-webhook', (req, res) => {
    const formData = req.body;

    if (formData.form_name === 'signup') {
        // Create trial account
        createTrialAccount({
            companyName: formData.companyName,
            firstName: formData.firstName,
            lastName: formData.lastName,
            email: formData.email,
            phone: formData.phone,
            techCount: formData.techCount,
            customerCount: formData.customerCount
        });
    }

    res.status(200).send('OK');
});
```

#### **B. Trial Account Creation**
```javascript
async function createTrialAccount(userData) {
    // 1. Create user account in your database
    const user = await User.create({
        email: userData.email,
        firstName: userData.firstName,
        lastName: userData.lastName,
        companyName: userData.companyName,
        phone: userData.phone,
        accountType: 'trial',
        trialStartDate: new Date(),
        trialEndDate: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000), // 30 days
        isActive: true
    });

    // 2. Generate temporary password
    const tempPassword = generateTempPassword();
    await user.setPassword(tempPassword);

    // 3. Send welcome email with login credentials
    await sendWelcomeEmail(user, tempPassword);

    // 4. Set up trial limitations
    await setupTrialLimitations(user);
}
```

---

## 🔐 **2. Trial Account Features & Limitations**

### **What Trial Users Get:**
- ✅ Full app access for 30 days
- ✅ Upload and manage up to 50 customers
- ✅ Send customer notifications
- ✅ Basic equipment tracking
- ✅ Route management
- ✅ Photo uploads and confirmations

### **Trial Limitations to Implement:**
- 🚫 Maximum 50 customers
- 🚫 Maximum 2 technicians
- 🚫 No custom branding
- 🚫 No advanced reporting
- 🚫 No API access
- 🚫 BlueView watermark on customer emails

### **Implementation Example:**
```javascript
// Middleware to check trial limitations
function checkTrialLimits(req, res, next) {
    const user = req.user;

    if (user.accountType === 'trial') {
        // Check customer limit
        if (req.path === '/customers' && req.method === 'POST') {
            const customerCount = await Customer.count({ userId: user.id });
            if (customerCount >= 50) {
                return res.status(403).json({
                    error: 'Trial limit reached',
                    message: 'Upgrade to add more customers',
                    upgradeUrl: '/upgrade'
                });
            }
        }

        // Check technician limit
        if (req.path === '/technicians' && req.method === 'POST') {
            const techCount = await Technician.count({ userId: user.id });
            if (techCount >= 2) {
                return res.status(403).json({
                    error: 'Trial limit reached',
                    message: 'Upgrade to add more technicians'
                });
            }
        }
    }

    next();
}
```

---

## 📱 **3. Mobile App Integration**

### **Trial User Experience:**
1. **Login Screen**: Trial users use email + temp password
2. **Trial Banner**: Show trial status and days remaining
3. **Upgrade Prompts**: Strategic upgrade CTAs throughout app
4. **Limitation Notices**: Clear messaging when hitting limits

### **Implementation:**
```javascript
// Check trial status on app load
async function checkTrialStatus(userId) {
    const user = await User.findById(userId);
    const now = new Date();

    if (user.accountType === 'trial') {
        const daysRemaining = Math.ceil((user.trialEndDate - now) / (1000 * 60 * 60 * 24));

        if (daysRemaining <= 0) {
            // Trial expired
            return {
                status: 'expired',
                message: 'Your trial has ended. Upgrade to continue using BlueView.',
                upgradeUrl: '/upgrade'
            };
        }

        return {
            status: 'active',
            daysRemaining: daysRemaining,
            message: `${daysRemaining} days left in your trial`
        };
    }

    return { status: 'paid' };
}
```

---

## 💳 **4. Payment Integration (Stripe Recommended)**

### **Upgrade Flow:**
1. User clicks "Upgrade" in app
2. Redirect to Stripe Checkout
3. On successful payment → convert trial to paid
4. Send confirmation email
5. Remove trial limitations

### **Stripe Integration:**
```javascript
// Create Stripe checkout session
app.post('/create-checkout-session', async (req, res) => {
    const user = req.user;

    const session = await stripe.checkout.sessions.create({
        customer_email: user.email,
        payment_method_types: ['card'],
        line_items: [{
            price: 'price_1J...', // Your Stripe price ID
            quantity: 1,
        }],
        mode: 'subscription',
        success_url: `${DOMAIN}/upgrade-success?session_id={CHECKOUT_SESSION_ID}`,
        cancel_url: `${DOMAIN}/pricing`,
        metadata: {
            userId: user.id
        }
    });

    res.json({ url: session.url });
});

// Handle successful payment
app.post('/stripe-webhook', (req, res) => {
    const event = req.body;

    if (event.type === 'checkout.session.completed') {
        const session = event.data.object;
        const userId = session.metadata.userId;

        // Upgrade trial to paid account
        upgradeTrialToPaid(userId, session);
    }

    res.status(200).send('OK');
});
```

---

## 📧 **5. Email Automation Sequence**

### **Trial Onboarding Emails:**

#### **Day 0: Welcome Email**
```
Subject: Welcome to BlueView! Your account is ready 🎉

Hi [FirstName],

Welcome to BlueView! Your 30-day trial has started.

Your login details:
- Email: [Email]
- Temporary Password: [TempPassword]
- Download: [App Store Link] | [Google Play Link]

Getting Started:
1. Download the mobile app
2. Add your first 5 customers
3. Complete your first route

Questions? Reply to this email.

Best regards,
The BlueView Team
```

#### **Day 7: Check-in Email**
```
Subject: How's your first week with BlueView going?

Hi [FirstName],

You're 1 week into your BlueView trial! Here's how other pool companies are using BlueView:

✅ Average 4x increase in customer satisfaction
✅ 50% reduction in "did you come" calls
✅ $200+ monthly revenue increase per technician

Need help? Book a 15-minute setup call: [Calendly Link]

23 days left in your trial.

[Upgrade Now Button]
```

#### **Day 20: Upgrade Reminder**
```
Subject: 10 days left in your BlueView trial

Hi [FirstName],

You have 10 days left in your trial. Don't lose access to:

• All your customer data
• Service history and photos
• Equipment tracking records

Upgrade now and get 20% off your first 3 months.

[Upgrade Now - 20% Off]

Questions? Just reply to this email.
```

#### **Day 30: Trial Ending**
```
Subject: Your BlueView trial ends today

Hi [FirstName],

Your trial access ends in 24 hours.

To keep using BlueView and retain your data:
[Upgrade Now - Save Your Data]

After tomorrow, trial accounts are deactivated and data is removed after 7 days.

Need more time to decide? Reply and we'll extend your trial.
```

---

## 🔄 **6. Backend API Endpoints Needed**

### **Essential Endpoints:**
```javascript
// Trial management
GET  /api/trial/status         // Check trial status
POST /api/trial/extend         // Extend trial (admin)
POST /api/trial/upgrade        // Upgrade to paid

// User management
POST /api/users/trial          // Create trial account
PUT  /api/users/password-reset // Reset temp password
GET  /api/users/limitations    // Get trial limitations

// Billing
POST /api/billing/checkout     // Create Stripe session
POST /api/billing/webhook      // Handle Stripe webhooks
GET  /api/billing/status       // Check payment status
```

---

## 📊 **7. Database Schema Updates**

### **Users Table Additions:**
```sql
ALTER TABLE users ADD COLUMN account_type VARCHAR(20) DEFAULT 'trial';
ALTER TABLE users ADD COLUMN trial_start_date TIMESTAMP;
ALTER TABLE users ADD COLUMN trial_end_date TIMESTAMP;
ALTER TABLE users ADD COLUMN stripe_customer_id VARCHAR(100);
ALTER TABLE users ADD COLUMN subscription_status VARCHAR(20);
ALTER TABLE users ADD COLUMN plan_type VARCHAR(20);
```

### **Trial Limitations Table:**
```sql
CREATE TABLE trial_limitations (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    max_customers INTEGER DEFAULT 50,
    max_technicians INTEGER DEFAULT 2,
    features_enabled JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 🔧 **8. Technical Architecture**

### **Recommended Tech Stack:**
- **Backend**: Node.js + Express (or Python + Django)
- **Database**: PostgreSQL
- **Payments**: Stripe
- **Email**: SendGrid or ConvertKit
- **Hosting**: Vercel/Heroku + AWS RDS

### **Data Flow:**
```
Website Form → Netlify → Webhook → Your API → Database
     ↓
Email Service → Welcome Email → User Downloads App
     ↓
Trial Account → 30 Days → Upgrade Prompts → Stripe → Paid Account
```

---

## 🚀 **9. Quick Implementation Steps**

### **Week 1: Basic Trial System**
1. Set up Netlify webhook endpoint
2. Create trial account creation logic
3. Add trial limitations to your app
4. Send welcome emails with login credentials

### **Week 2: Payment Integration**
1. Set up Stripe account and products
2. Create checkout flow
3. Handle successful payments
4. Implement account upgrades

### **Week 3: Email Automation**
1. Set up email service (SendGrid/ConvertKit)
2. Create email templates
3. Set up automated sequences
4. Add trial expiration handling

### **Week 4: Polish & Testing**
1. Test complete trial flow
2. Add trial status indicators in app
3. Optimize upgrade prompts
4. Launch and monitor conversions

---

## 📈 **10. Conversion Optimization**

### **Key Metrics to Track:**
- **Trial Signup Rate**: Website visitors → Trial signups
- **Activation Rate**: Trial signups → Active users (completed first route)
- **Trial-to-Paid Conversion**: Trial users → Paying customers
- **Time to First Value**: Days until user completes first service

### **Optimization Strategies:**
1. **Reduce friction**: Fewer form fields, automatic setup
2. **Quick wins**: Help users complete first route within 24 hours
3. **Value demonstration**: Show ROI calculations in app
4. **Strategic prompts**: Upgrade prompts at high-value moments
5. **Personal touch**: Call high-value prospects during trial

---

## 💡 **11. Pro Tips for Success**

### **Trial Best Practices:**
1. **Onboard immediately**: Call/email within 24 hours
2. **Show value fast**: Help complete first route ASAP
3. **Track engagement**: Monitor who's actively using features
4. **Proactive support**: Reach out to inactive users
5. **Social proof**: Share success stories during trial

### **Common Pitfalls to Avoid:**
- ❌ Making trial signup too complicated
- ❌ Not following up with new trial users
- ❌ Waiting until trial end to ask for upgrade
- ❌ Not showing clear value during trial
- ❌ Making upgrade process difficult

---

## 🔧 **12. Example Code Repository Structure**

```
/backend
  /routes
    - webhooks.js       # Netlify webhooks
    - auth.js          # Trial authentication
    - billing.js       # Stripe integration
    - trial.js         # Trial management
  /models
    - User.js          # User model with trial fields
    - Subscription.js  # Subscription tracking
  /middleware
    - trialLimits.js   # Trial limitation checks
  /services
    - emailService.js  # Email automation
    - stripeService.js # Payment processing

/mobile-app
  /components
    - TrialBanner.js   # Shows trial status
    - UpgradePrompt.js # Upgrade CTAs
  /screens
    - TrialStatus.js   # Trial management screen
    - Upgrade.js       # Subscription upgrade
```

---

This guide gives you everything needed to implement a complete trial system that converts website visitors into paying customers. Start with the basic trial creation and work through each step systematically.

**Key Success Factor**: The faster you can get trial users to their first "aha moment" (completing their first route and seeing customer satisfaction), the higher your conversion rate will be.