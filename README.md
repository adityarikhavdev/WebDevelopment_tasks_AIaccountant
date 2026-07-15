What is this task?
Two important flows are covered here. Email Verification — when a new user registers, they receive a confirmation link. Only after clicking it is their account activated. Password Reset — when a user forgets their password, a unique time-limited link is sent to their email allowing them to set a new one. Both flows use the same core concept — generate a secret token, store it with an expiry time, send it via email, validate it when clicked.
________________________________________
What exactly needs to be built?
1. Update the Database Schema (Backend — Prisma)
Add verification and reset fields to the User model:
js
model User {
  // ... existing fields
  emailVerified     Boolean   @default(false)
  verifyToken       String?   @unique
  verifyTokenExpiry DateTime?
  resetToken        String?   @unique
  resetTokenExpiry  DateTime?
}
Run npx prisma migrate dev --name add-email-reset-fields.
________________________________________
2. Set Up Email Sending (Backend — Node.js)
Install Resend for sending emails:
bash
npm install resend
Create lib/email.js:
js
const { Resend } = require('resend')
const resend = new Resend(process.env.RESEND_API_KEY)

async function sendVerificationEmail(toEmail, userName, token) {
  const link = `${process.env.CLIENT_URL}/verify-email?token=${token}`
  await resend.emails.send({
    from: 'AI Accountant <noreply@yourdomain.com>',
    to: toEmail,
    subject: 'Verify your email address',
    html: `<p>Hi ${userName},</p>
           <p>Click the link below to verify your email. This link expires in 24 hours.</p>
           <a href="${link}">Verify My Email</a>`
  })
}

async function sendPasswordResetEmail(toEmail, userName, token) {
  const link = `${process.env.CLIENT_URL}/reset-password?token=${token}`
  await resend.emails.send({
    from: 'AI Accountant <noreply@yourdomain.com>',
    to: toEmail,
    subject: 'Reset your password',
    html: `<p>Hi ${userName},</p>
           <p>Click the link below to reset your password. This link expires in 1 hour.</p>
           <a href="${link}">Reset Password</a>`
  })
}

module.exports = { sendVerificationEmail, sendPasswordResetEmail }
Add RESEND_API_KEY and CLIENT_URL=http://localhost:3000 to your .env file.
________________________________________
3. Email Verification — Backend Routes (Node.js Express)
In routes/auth.js, after creating a user during registration, generate and send a verification token:
js
const crypto = require('crypto')
const { sendVerificationEmail } = require('../lib/email')

// Called inside POST /register after user is created
const verifyToken = crypto.randomBytes(32).toString('hex')
const verifyTokenExpiry = new Date(Date.now() + 24 * 60 * 60 * 1000) // 24hrs

await prisma.user.update({
  where: { id: newUser.id },
  data: { verifyToken, verifyTokenExpiry }
})

await sendVerificationEmail(newUser.email, newUser.name, verifyToken)
Create POST /api/auth/verify-email route:
js
router.post('/verify-email', async (req, res) => {
  const { token } = req.body

  const user = await prisma.user.findUnique({
    where: { verifyToken: token }
  })

  if (!user) return res.status(400).json({ error: 'Invalid link' })

  if (user.verifyTokenExpiry < new Date()) {
    return res.status(400).json({ error: 'Link has expired' })
  }

  await prisma.user.update({
    where: { id: user.id },
    data: {
      emailVerified: true,
      verifyToken: null,
      verifyTokenExpiry: null
    }
  })

  res.json({ message: 'Email verified successfully' })
})
Block login for unverified users — add this check inside POST /login:
js
if (!user.emailVerified) {
  return res.status(400).json({
    error: 'Please verify your email before logging in.'
  })
}
________________________________________
4. Password Reset — Backend Routes (Node.js Express)
Create POST /api/auth/forgot-password:
js
router.post('/forgot-password', async (req, res) => {
  const { email } = req.body
  const user = await prisma.user.findUnique({ where: { email } })

  // Always return same message — never reveal if email exists
  if (!user) {
    return res.json({ message: 'If this email exists, a reset link has been sent.' })
  }

  const resetToken = crypto.randomBytes(32).toString('hex')
  const resetTokenExpiry = new Date(Date.now() + 60 * 60 * 1000) // 1 hour

  await prisma.user.update({
    where: { id: user.id },
    data: { resetToken, resetTokenExpiry }
  })

  await sendPasswordResetEmail(user.email, user.name, resetToken)
  res.json({ message: 'If this email exists, a reset link has been sent.' })
})
Create POST /api/auth/reset-password:
js
router.post('/reset-password', async (req, res) => {
  const { token, newPassword } = req.body

  const user = await prisma.user.findUnique({ where: { resetToken: token } })

  if (!user) return res.status(400).json({ error: 'Invalid reset link' })
  if (user.resetTokenExpiry < new Date()) {
    return res.status(400).json({ error: 'Reset link has expired' })
  }
  if (newPassword.length < 8) {
    return res.status(400).json({ error: 'Password must be at least 8 characters' })
  }

  const hashed = await bcrypt.hash(newPassword, 12)

  await prisma.user.update({
    where: { id: user.id },
    data: {
      password: hashed,
      resetToken: null,
      resetTokenExpiry: null
    }
  })

  res.json({ message: 'Password reset successfully. You can now log in.' })
})
________________________________________
5. Frontend Pages (React.js)
Create four pages using React Router:
/verify-email — Reads the token query param from the URL using useSearchParams, calls POST /api/auth/verify-email, shows success or error message, redirects to /login after 3 seconds.
/forgot-password — A single email input form. On submit, calls POST /api/auth/forgot-password. Always shows the same success message regardless of whether the email exists.
/reset-password — Reads token from URL, shows New Password and Confirm Password fields. On submit, calls POST /api/auth/reset-password. On success, redirects to /login.
Add a Forgot Password? link below the password field on the Login page pointing to /forgot-password.
________________________________________
Key things to remember
•	Delete tokens after use — set resetToken: null immediately after it is used. A reusable reset token is a serious security hole
•	Always use the same response message for forgot password — never say "No account found" as it reveals which emails are registered
•	Hash the new password — the same bcryptjs hashing as registration. Never store plain text
•	1 hour expiry for reset, 24 hours for verification — these are the industry standards
