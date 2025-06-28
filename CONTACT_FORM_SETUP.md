# Contact Form 7 Integration Setup Guide

## Current Status
- **Frontend**: Astro site at https://maxsys.org
- **Backend**: WordPress at https://give.maxsys.org
- **Contact Form ID**: 2927 (Contact form 1)
- **Issue**: Contact form not working - needs proper CF7 REST API setup

## Step 1: WordPress Backend Setup (https://give.maxsys.org)

### 1.1 Install Required Plugins
Log into your WordPress admin at `https://give.maxsys.org/wp-admin` and ensure these plugins are installed and activated:

1. **Contact Form 7** (should already be installed)
2. **Contact Form 7 REST API** - Install this plugin:
   - Go to Plugins → Add New
   - Search for "Contact Form 7 REST API"
   - Install and activate the plugin by Takayuki Miyauchi

### 1.2 Verify Contact Form Configuration
1. Go to **Contact → Contact Forms**
2. Find "Contact form 1" (ID: 2927)
3. Click **Edit**
4. Ensure the form has these fields with exact names:
   ```
   [text* your-name placeholder "Your Name"]
   [email* your-email placeholder "Your Email"]
   [text* your-subject placeholder "Subject"]
   [textarea* your-message placeholder "Your Message"]
   [submit "Send Message"]
   ```

### 1.3 Configure CORS Headers
Add this code to your WordPress theme's `functions.php` file or create a custom plugin:

```php
// Enable CORS for Contact Form 7 REST API
add_action('rest_api_init', function() {
    remove_filter('rest_pre_serve_request', 'rest_send_cors_headers');
    add_filter('rest_pre_serve_request', function($value) {
        header('Access-Control-Allow-Origin: https://maxsys.org');
        header('Access-Control-Allow-Methods: POST, GET, OPTIONS');
        header('Access-Control-Allow-Headers: Content-Type, Authorization');
        header('Access-Control-Allow-Credentials: true');
        
        if ('OPTIONS' === $_SERVER['REQUEST_METHOD']) {
            status_header(200);
            exit();
        }
        
        return $value;
    });
});

// Specifically for Contact Form 7 endpoints
add_filter('rest_pre_serve_request', function($served, $result, $request, $server) {
    if (strpos($request->get_route(), '/contact-form-7/') !== false) {
        header('Access-Control-Allow-Origin: https://maxsys.org');
        header('Access-Control-Allow-Methods: POST, OPTIONS');
        header('Access-Control-Allow-Headers: Content-Type');
    }
    return $served;
}, 10, 4);
```

### 1.4 Test the API Endpoint
Test the endpoint manually using curl or Postman:

```bash
curl -X POST https://give.maxsys.org/wp-json/contact-form-7/v1/contact-forms/2927/feedback \
  -H "Content-Type: multipart/form-data" \
  -F "your-name=Test User" \
  -F "your-email=test@example.com" \
  -F "your-subject=Test Subject" \
  -F "your-message=Test message content"
```

Expected response:
```json
{
  "status": "mail_sent",
  "message": "Thank you for your message. It has been sent."
}
```

## Step 2: Alternative Setup (If CF7 REST API doesn't work)

### 2.1 Create Custom WordPress Endpoint
If the CF7 REST API plugin doesn't work, add this custom endpoint to your `functions.php`:

```php
// Custom contact form endpoint
add_action('rest_api_init', function() {
    register_rest_route('maxsys/v1', '/contact', array(
        'methods' => 'POST',
        'callback' => 'handle_contact_form_submission',
        'permission_callback' => '__return_true'
    ));
});

function handle_contact_form_submission($request) {
    // Enable CORS
    header('Access-Control-Allow-Origin: https://maxsys.org');
    header('Access-Control-Allow-Methods: POST, OPTIONS');
    header('Access-Control-Allow-Headers: Content-Type');
    
    $params = $request->get_params();
    
    // Validate required fields
    if (empty($params['name']) || empty($params['email']) || empty($params['subject']) || empty($params['message'])) {
        return new WP_Error('missing_fields', 'All fields are required', array('status' => 400));
    }
    
    // Sanitize input
    $name = sanitize_text_field($params['name']);
    $email = sanitize_email($params['email']);
    $subject = sanitize_text_field($params['subject']);
    $message = sanitize_textarea_field($params['message']);
    
    // Validate email
    if (!is_email($email)) {
        return new WP_Error('invalid_email', 'Invalid email address', array('status' => 400));
    }
    
    // Send email
    $to = get_option('admin_email'); // or your specific email
    $email_subject = 'Contact Form: ' . $subject;
    $email_message = "Name: $name\n";
    $email_message .= "Email: $email\n";
    $email_message .= "Subject: $subject\n\n";
    $email_message .= "Message:\n$message";
    
    $headers = array(
        'Content-Type: text/plain; charset=UTF-8',
        'From: ' . $name . ' <' . $email . '>',
        'Reply-To: ' . $email
    );
    
    $sent = wp_mail($to, $email_subject, $email_message, $headers);
    
    if ($sent) {
        return array(
            'status' => 'success',
            'message' => 'Thank you for your message. It has been sent.'
        );
    } else {
        return new WP_Error('send_failed', 'Failed to send message', array('status' => 500));
    }
}
```

## Step 3: Frontend Environment Variables

Your Cloudflare environment variables look correct. The contact form doesn't need the API keys - those are for donations and reCAPTCHA.

## Step 4: Testing Checklist

### Backend Tests:
1. ✅ Contact Form 7 plugin installed and activated
2. ✅ Contact Form 7 REST API plugin installed and activated
3. ✅ Form ID 2927 exists and has correct field names
4. ✅ CORS headers configured
5. ✅ API endpoint responds correctly

### Frontend Tests:
1. ✅ Form displays correctly
2. ✅ Validation works
3. ✅ Submission shows loading state
4. ✅ Success/error messages display
5. ✅ Form resets after successful submission

## Step 5: Troubleshooting

### Common Issues:

1. **CORS Errors**: Ensure CORS headers are properly configured in WordPress
2. **404 on API endpoint**: Verify Contact Form 7 REST API plugin is activated
3. **Form ID mismatch**: Double-check the form ID is 2927
4. **Field name mismatch**: Ensure WordPress form uses `your-name`, `your-email`, etc.
5. **Email not sending**: Check WordPress email configuration

### Debug Steps:

1. Check browser console for errors
2. Check WordPress error logs
3. Test API endpoint directly with curl
4. Verify form field names match exactly
5. Check if emails are being sent (check spam folder)

## Step 6: Alternative Solutions

If Contact Form 7 continues to have issues, consider:

1. **WPForms** with REST API
2. **Gravity Forms** with REST API
3. **Custom WordPress endpoint** (provided above)
4. **Third-party service** like Formspree or Netlify Forms

## Support

If you continue to have issues:
1. Check WordPress admin for Contact Form 7 logs
2. Enable WordPress debug logging
3. Test the API endpoint manually
4. Verify all plugins are up to date