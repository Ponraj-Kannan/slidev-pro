<script setup>
import { ref, computed, onMounted, onUnmounted, nextTick, watchEffect } from 'vue'
import { authState, logout } from '../auth'

// Sync auth state with DOM body/html classes to hide/show slides securely
watchEffect(() => {
  if (typeof document !== 'undefined') {
    if (authState.isLoggedIn) {
      document.documentElement.classList.add('auth-success')
    } else {
      document.documentElement.classList.remove('auth-success')
    }
  }
})

const errorMessage = ref('')
const successMessage = ref('')
const newEmail = ref('')
const isLoading = ref(false)
const isPageLoading = ref(true)

// Real-time email parser for auto-separation
const parsedEmailsPreview = computed(() => {
  if (!newEmail.value) return []
  // Matches any email-like strings globally
  const emailRegexGlobal = /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g
  const matches = newEmail.value.match(emailRegexGlobal) || []
  // Deduplicate and trim
  return [...new Set(matches.map(e => e.trim().toLowerCase()))]
})

const isNotAllowed = ref(false)
const notAllowedEmail = ref('')
const isAdminPage = ref(false)

function checkIsAdminPage() {
  if (typeof window !== 'undefined') {
    const isPathAdmin = window.location.pathname === '/admin' || window.location.pathname === '/admin.html'
    const isQueryAdmin = window.location.search.includes('admin')
    const isHashAdmin = window.location.hash === '#admin'
    isAdminPage.value = isPathAdmin || isQueryAdmin || isHashAdmin
  }
}

function resetLogin() {
  isNotAllowed.value = false
  notAllowedEmail.value = ''
  errorMessage.value = ''
  initGoogleSignIn()
}

function closeAdminPanel() {
  if (isAdminPage.value) {
    window.location.href = '/'
  } else {
    authState.showAdminPanel = false
  }
}

// Helper to decode JWT on the client
function decodeJwt(token) {
  try {
    const base64Url = token.split('.')[1]
    const base64 = base64Url.replace(/-/g, '+').replace(/_/g, '/')
    const jsonPayload = decodeURIComponent(atob(base64).split('').map(function(c) {
      return '%' + ('00' + c.charCodeAt(0).toString(16)).slice(-2)
    }).join(''))
    return JSON.parse(jsonPayload)
  } catch (e) {
    console.error('Failed to decode JWT:', e)
    return null
  }
}

// Fetch whitelisted emails from API
async function fetchWhitelistedEmails() {
  try {
    const response = await fetch('/api/emails')
    if (response.ok) {
      const data = await response.json()
      authState.allowedEmails = data.emails || []
      localStorage.setItem('fp_allowed_emails', JSON.stringify(authState.allowedEmails))
    } else {
      throw new Error('API response not ok')
    }
  } catch (error) {
    console.warn('Could not fetch allowed emails from server. Falling back to local cache/default.', error)
    // Fallback: load from localStorage or seed
    const local = localStorage.getItem('fp_allowed_emails')
    if (local) {
      authState.allowedEmails = JSON.parse(local)
    } else {
      authState.allowedEmails = ['ponrajacc@gmail.com']
      localStorage.setItem('fp_allowed_emails', JSON.stringify(authState.allowedEmails))
    }
  }
}

// Check local storage for existing session
async function checkAuthSession() {
  checkIsAdminPage()
  isPageLoading.value = true
  await fetchWhitelistedEmails()

  const savedToken = localStorage.getItem('fp_auth_token')
  const savedEmail = localStorage.getItem('fp_auth_email')
  const savedName = localStorage.getItem('fp_auth_name')
  const savedPicture = localStorage.getItem('fp_auth_picture')

  if (savedToken && savedEmail) {
    // Check if the saved email is still allowed
    const isAllowed = authState.allowedEmails.map(e => e.toLowerCase()).includes(savedEmail.toLowerCase())
    if (isAllowed) {
      authState.isLoggedIn = true
      authState.userEmail = savedEmail
      authState.userName = savedName || ''
      authState.userPicture = savedPicture || ''
      authState.idToken = savedToken
      authState.isAdmin = savedEmail.toLowerCase() === 'ponrajacc@gmail.com'
    } else {
      // Whitelist changed, no longer allowed
      logout()
      errorMessage.value = `Session expired: Email '${savedEmail}' is no longer whitelisted.`
    }
  }
  isPageLoading.value = false
}

// Initialize Google Identity Services
function initGoogleSignIn() {
  if (authState.isLoggedIn) return

  // Render Google Button in Next Tick to ensure div is present
  nextTick(() => {
    try {
      if (typeof window.google === 'undefined') {
        console.error('Google Identity Services script not loaded')
        return
      }

      window.google.accounts.id.initialize({
        client_id: '207254417956-gidt8ji2ocn6fk88e0no3ogg2m46huig.apps.googleusercontent.com',
        callback: handleGoogleSignInCallback,
        auto_select: false
      })

      const btnEl = document.getElementById('google-signin-btn')
      if (btnEl) {
        window.google.accounts.id.renderButton(btnEl, {
          theme: 'filled_blue',
          size: 'large',
          width: '280',
          text: 'signin_with',
          shape: 'pill'
        })
      }
    } catch (err) {
      console.error('Failed to initialize Google Sign In:', err)
    }
  })
}

// Google Sign-In Callback
async function handleGoogleSignInCallback(response) {
  errorMessage.value = ''
  successMessage.value = ''
  
  if (!response.credential) {
    errorMessage.value = 'Failed to retrieve login credentials from Google.'
    return
  }

  const payload = decodeJwt(response.credential)
  if (!payload) {
    errorMessage.value = 'Failed to decode identity token.'
    return
  }

  const email = payload.email ? payload.email.toLowerCase() : ''
  const isEmailVerified = payload.email_verified === true || payload.email_verified === 'true'

  if (!email) {
    errorMessage.value = 'Google account did not return a valid email address.'
    return
  }

  if (!isEmailVerified) {
    errorMessage.value = 'Your Google email address is not verified.'
    return
  }

  // Reload the emails list right before validation to ensure fresh whitelist data
  await fetchWhitelistedEmails()

  const isAllowed = authState.allowedEmails.map(e => e.toLowerCase()).includes(email)

  if (isAllowed) {
    // Authenticated and whitelisted
    authState.isLoggedIn = true
    authState.userEmail = email
    authState.userName = payload.name || ''
    authState.userPicture = payload.picture || ''
    authState.idToken = response.credential
    authState.isAdmin = email === 'ponrajacc@gmail.com'

    // Persist session
    localStorage.setItem('fp_auth_token', response.credential)
    localStorage.setItem('fp_auth_email', email)
    localStorage.setItem('fp_auth_name', payload.name || '')
    localStorage.setItem('fp_auth_picture', payload.picture || '')
  } else {
    // Authenticated but NOT whitelisted
    isNotAllowed.value = true
    notAllowedEmail.value = email
  }
}

// Add new email(s) (Admin only)
async function addEmail() {
  const emailsToWhitelist = parsedEmailsPreview.value
  if (emailsToWhitelist.length === 0) return

  errorMessage.value = ''
  successMessage.value = ''
  isLoading.value = true

  try {
    const response = await fetch('/api/emails', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${authState.idToken}`
      },
      body: JSON.stringify({ emails: emailsToWhitelist })
    })

    const data = await response.json()

    if (response.ok) {
      authState.allowedEmails = data.emails || []
      localStorage.setItem('fp_allowed_emails', JSON.stringify(authState.allowedEmails))
      successMessage.value = data.message || `Successfully whitelisted ${emailsToWhitelist.length} email(s)`
      newEmail.value = ''
    } else {
      throw new Error(data.error || 'Failed to add emails to whitelist')
    }
  } catch (error) {
    console.error('Failed to add emails to whitelist:', error)
    errorMessage.value = error.message || 'Failed to add emails to whitelist'
  } finally {
    isLoading.value = false
  }
}

// Delete email (Admin only)
async function removeEmail(emailToRemove) {
  if (emailToRemove === 'ponrajacc@gmail.com') {
    errorMessage.value = 'Cannot remove the primary administrator email.'
    return
  }

  if (!confirm(`Are you sure you want to remove '${emailToRemove}' from the whitelist?`)) {
    return
  }

  errorMessage.value = ''
  successMessage.value = ''
  isLoading.value = true

  try {
    const response = await fetch('/api/emails', {
      method: 'DELETE',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${authState.idToken}`
      },
      body: JSON.stringify({ email: emailToRemove })
    })

    const data = await response.json()

    if (response.ok) {
      authState.allowedEmails = data.emails || []
      localStorage.setItem('fp_allowed_emails', JSON.stringify(authState.allowedEmails))
      successMessage.value = data.message || `Removed successfully: ${emailToRemove}`
    } else {
      throw new Error(data.error || 'Failed to remove email')
    }
  } catch (error) {
    console.error('Failed to remove email:', error)
    errorMessage.value = error.message || 'Failed to remove email'
  } finally {
    isLoading.value = false
  }
}

// Block Slidev keyboard navigation keys when not logged in
function blockKeyboard(e) {
  if (!authState.isLoggedIn) {
    const blockedKeys = ['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight', 'PageUp', 'PageDown', 'Home', 'End', ' ', 'Spacebar', 'Enter']
    if (blockedKeys.includes(e.key)) {
      if (e.target.tagName === 'INPUT' || e.target.tagName === 'TEXTAREA') {
        return
      }
      e.preventDefault()
      e.stopImmediatePropagation()
    }
  }
}

onMounted(() => {
  window.addEventListener('keydown', blockKeyboard, { capture: true })
  window.addEventListener('keyup', blockKeyboard, { capture: true })
  window.addEventListener('keypress', blockKeyboard, { capture: true })

  checkAuthSession().then(() => {
    // Dynamic Google Client Script Load if not loaded yet
    if (typeof window.google === 'undefined') {
      const script = document.createElement('script')
      script.src = 'https://accounts.google.com/gsi/client'
      script.async = true
      script.defer = true
      script.onload = () => initGoogleSignIn()
      script.onerror = () => {
        errorMessage.value = 'Could not load Google Sign-In SDK. Check your internet connection.'
      }
      document.head.appendChild(script)
    } else {
      initGoogleSignIn()
    }
  })
})

onUnmounted(() => {
  window.removeEventListener('keydown', blockKeyboard, { capture: true })
  window.removeEventListener('keyup', blockKeyboard, { capture: true })
  window.removeEventListener('keypress', blockKeyboard, { capture: true })
})
</script>

<template>
  <div>
    <!-- 1. FULL-SCREEN LOGIN OVERLAY (shown if not logged in) -->
    <Transition name="fade">
      <div v-if="!authState.isLoggedIn && !isPageLoading" class="login-overlay">
        <!-- Visual background glowing elements -->
        <div class="blob blob-orange"></div>
        <div class="blob blob-purple"></div>
        
        <div class="login-card">
          <!-- Brand header -->
          <div class="brand-container">
            <div class="brand-logo">
              <img  src="../assets/logo.png" style="width: 150px;"/>
            </div>
            <p class="brand-tagline">{{ isAdminPage ? 'Admin Whitelist Portal' : 'Interactive Slide Deck Portal' }}</p>
          </div>

          <div class="divider"></div>

          <!-- Card Body -->
          <div class="login-body">
            <template v-if="isNotAllowed">
              <div class="not-allowed-icon-container">
                <svg class="not-allowed-large-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                  <circle cx="12" cy="12" r="10"/>
                  <line x1="15" y1="9" x2="9" y2="15"/>
                  <line x1="9" y1="9" x2="15" y2="15"/>
                </svg>
              </div>
              <h2 class="login-heading text-red">Not allowed</h2>
              <p class="login-subtext text-center">
                This email is not whitelisted to access the presentation.
              </p>
              
              <div class="not-allowed-actions">
                <button @click="resetLogin" class="action-btn-retry width-full">Try Another Account</button>
              </div>
            </template>

            <template v-else>
              <h2 class="login-heading">{{ isAdminPage ? 'Admin Sign In' : 'Sign In Required' }}</h2>
              <p class="login-subtext">
                {{ isAdminPage 
                  ? 'Sign in to the Whitelist Management portal using an administrator Google account.' 
                  : 'Access to this interactive slide deck is restricted. Please sign in with a whitelisted Google Account.' }}
              </p>
              
              <!-- Google Button Container -->
              <div class="signin-button-wrapper">
                <div id="google-signin-btn"></div>
              </div>

              <!-- Error Banner -->
              <Transition name="slide-up">
                <div v-if="errorMessage" class="error-banner">
                  <svg class="error-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                    <path d="M10.29 3.86L1.82 18a2 2 0 0 0 1.71 3h16.94a2 2 0 0 0 1.71-3L13.71 3.86a2 2 0 0 0-3.42 0z"/>
                    <line x1="12" y1="9" x2="12" y2="13"/>
                    <line x1="12" y1="17" x2="12.01" y2="17"/>
                  </svg>
                  <div class="error-text">{{ errorMessage }}</div>
                </div>
              </Transition>
            </template>
          </div>
        </div>
      </div>
    </Transition>

    <!-- Page Loading Indicator -->
    <div v-if="isPageLoading" class="page-loader-overlay">
      <div class="loader-spinner"></div>
      <p class="loader-text">Loading deck authorization...</p>
    </div>

    <!-- 2. ADMIN DASHBOARD MODAL/PAGE (shown to admin on toggle or direct route) -->
    <Transition name="fade">
      <div v-if="authState.isLoggedIn && authState.isAdmin && (authState.showAdminPanel || isAdminPage)" class="admin-modal-overlay" @click.self="closeAdminPanel">
        <div class="admin-card">
          <!-- Header -->
          <div class="admin-header">
            <div class="admin-header-title">
              <svg class="admin-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M12.22 2h-.44a2 2 0 0 0-2 2v.18a2 2 0 0 1-1 1.73l-.43.25a2 2 0 0 1-2 0l-.15-.08a2 2 0 0 0-2.73.73l-.22.38a2 2 0 0 0 .73 2.73l.15.1a2 2 0 0 1 1 1.72v.51a2 2 0 0 1-1 1.74l-.15.09a2 2 0 0 0-.73 2.73l.22.38a2 2 0 0 0 2.73.73l.15-.08a2 2 0 0 1 2 0l.43.25a2 2 0 0 1 1 1.73V20a2 2 0 0 0 2 2h.44a2 2 0 0 0 2-2v-.18a2 2 0 0 1 1-1.73l.43-.25a2 2 0 0 1 2 0l.15.08a2 2 0 0 0 2.73-.73l.22-.39a2 2 0 0 0-.73-2.73l-.15-.08a2 2 0 0 1-1-1.74v-.5a2 2 0 0 1 1-1.74l.15-.1a2 2 0 0 0 .73-2.73l-.22-.38a2 2 0 0 0-2.73-.73l-.15.08a2 2 0 0 1-2 0l-.43-.25a2 2 0 0 1-1-1.73V4a2 2 0 0 0-2-2z"/>
                <circle cx="12" cy="12" r="3"/>
              </svg>
              <h2>Whitelist Email Management</h2>
            </div>
            <button class="close-btn" @click="closeAdminPanel" :title="isAdminPage ? 'Back to Slides' : 'Close'">
              <svg v-if="isAdminPage" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" style="width: 20px; height: 20px;">
                <path d="M3 9l9-7 9 7v11a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2z"/>
                <polyline points="9 22 9 12 15 12 15 22"/>
              </svg>
              <svg v-else viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <line x1="18" y1="6" x2="6" y2="18"/>
                <line x1="6" y1="6" x2="18" y2="18"/>
              </svg>
            </button>
          </div>

          <!-- Body -->
          <div class="admin-body">
            <!-- Alert Banners inside Modal -->
            <div v-if="errorMessage" class="error-banner mb-3">
              <span>{{ errorMessage }}</span>
            </div>
            <div v-if="successMessage" class="success-banner mb-3">
              <svg class="success-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M22 11.08V12a10 10 0 1 1-5.93-9.14"/>
                <polyline points="22 4 12 14.01 9 11.01"/>
              </svg>
              <span>{{ successMessage }}</span>
            </div>

            <!-- Add Email Form -->
            <form @submit.prevent="addEmail" class="add-form">
              <textarea 
                v-model="newEmail" 
                placeholder="Paste text containing email addresses (e.g. copied from Excel, a document, or email list)" 
                required 
                class="form-input form-textarea"
                rows="3"
                :disabled="isLoading"
              ></textarea>

              <!-- Live Auto-Separated Emails Preview -->
              <div v-if="parsedEmailsPreview.length > 0" class="parsed-preview-box">
                <span class="preview-title">Detected {{ parsedEmailsPreview.length }} email(s):</span>
                <div class="preview-tags">
                  <span v-for="email in parsedEmailsPreview" :key="email" class="preview-tag">
                    {{ email }}
                  </span>
                </div>
              </div>

              <button type="submit" class="submit-btn" :disabled="isLoading || parsedEmailsPreview.length === 0">
                <span v-if="isLoading" class="btn-spinner"></span>
                <span v-else>Whitelist {{ parsedEmailsPreview.length }} Email(s)</span>
              </button>
            </form>

            <div class="whitelisted-heading">
              <span>Whitelisted Accounts ({{ authState.allowedEmails.length }})</span>
            </div>

            <!-- Whitelist Table / List -->
            <div class="emails-list-container">
              <div v-if="authState.allowedEmails.length === 0" class="no-emails">
                No whitelisted emails found.
              </div>
              <ul v-else class="emails-list">
                <li v-for="email in authState.allowedEmails" :key="email" class="email-item">
                  <div class="email-details">
                    <span class="email-text">{{ email }}</span>
                    <span v-if="email === 'ponrajacc@gmail.com'" class="badge-admin">Primary Admin</span>
                  </div>
                  <button 
                    v-if="email !== 'ponrajacc@gmail.com'"
                    @click="removeEmail(email)" 
                    class="delete-btn" 
                    title="Remove access"
                    :disabled="isLoading"
                  >
                    <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" class="trash-icon">
                      <polyline points="3 6 5 6 21 6"/>
                      <path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"/>
                      <line x1="10" y1="11" x2="10" y2="17"/>
                      <line x1="14" y1="11" x2="14" y2="17"/>
                    </svg>
                  </button>
                </li>
              </ul>
            </div>
          </div>
        </div>
      </div>
    </Transition>

    <!-- 3. ADMIN ACCESS DENIED (shown if logged in but not admin on admin page) -->
    <Transition name="fade">
      <div v-if="isAdminPage && authState.isLoggedIn && !authState.isAdmin" class="login-overlay">
        <div class="login-card">
          <div class="brand-container">
            <div class="brand-logo">
              <img  src="../assets/logo.png" style="width: 150px;"/>
            </div>
            <p class="brand-tagline">Admin Portal</p>
          </div>
          <div class="divider"></div>
          <div class="login-body">
            <div class="not-allowed-icon-container text-red">
              <svg class="not-allowed-large-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <rect x="3" y="11" width="18" height="11" rx="2" ry="2"/>
                <path d="M7 11V7a5 5 0 0 1 10 0v4"/>
              </svg>
            </div>
            <h2 class="login-heading text-red">Access Denied</h2>
            <p class="login-subtext text-center">Administrator privileges are required to access this portal. Currently signed in as <strong>{{ authState.userEmail }}</strong>.</p>
            <div class="not-allowed-actions">
              <button @click="logout" class="action-btn-retry width-full">Sign Out / Switch Account</button>
              <a href="/" class="action-btn-admin width-full text-center">Back to Slides</a>
            </div>
          </div>
        </div>
      </div>
    </Transition>
  </div>
</template>

<style scoped>
/* ── Simplified Access Denied & Admin Portal Styles ────── */
.text-red {
  color: #ef4444 !important;
}

.text-center {
  text-align: center;
}

.width-full {
  width: 100% !important;
}

.not-allowed-icon-container {
  display: flex;
  justify-content: center;
  margin-bottom: 1.25rem;
  color: #ef4444;
}

.not-allowed-large-icon {
  width: 56px;
  height: 56px;
}

.not-allowed-actions {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  margin-top: 1.5rem;
  width: 100%;
}

.action-btn-admin {
  background: #ef5050;
  color: white !important;
  text-decoration: none;
  font-weight: 600;
  font-size: 0.88rem;
  padding: 0.75rem 1.25rem;
  border-radius: 8px;
  text-align: center;
  transition: background 0.2s ease;
  display: block;
  border: none;
  cursor: pointer;
  box-sizing: border-box;
}

.action-btn-admin:hover {
  background: #db3b3b;
}

.action-btn-retry {
  background: #ffffff;
  color: #475569 !important;
  border: 1px solid #cbd5e1;
  font-weight: 600;
  font-size: 0.88rem;
  padding: 0.75rem 1.25rem;
  border-radius: 8px;
  text-align: center;
  transition: all 0.2s ease;
  cursor: pointer;
  box-sizing: border-box;
}

.action-btn-retry:hover {
  background: #f1f5f9;
  color: #0f172a !important;
  border-color: #94a3b8;
}

.admin-link {
  color: #ef5050 !important;
  text-decoration: underline;
  font-weight: 600;
  display: inline-block;
  margin-top: 0.25rem;
  transition: color 0.2s ease;
}

.admin-link:hover {
  color: #f87171 !important;
}

/* ── Layout & Background ────────────────────────────────── */
.login-overlay {
  position: fixed;
  inset: 0;
  width: 100vw;
  height: 76vh;
  /* background: #7491ad; */
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 999999;
  font-family: 'Inter', system-ui, -apple-system, sans-serif;
  overflow: hidden;
}

.page-loader-overlay {
  position: fixed;
  inset: 0;
  width: 100vw;
  height: 100vh;
  background: #f8fafc;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  z-index: 999998;
  font-family: 'Inter', system-ui, sans-serif;
}

.loader-spinner {
  width: 48px;
  height: 48px;
  border: 4px solid rgba(239, 80, 80, 0.15);
  border-left-color: #ef5050;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin-bottom: 1rem;
}

.loader-text {
  color: #64748b;
  font-size: 0.95rem;
}

/* Glowing blur blobs */
.blob {
  position: absolute;
  width: 400px;
  height: 400px;
  border-radius: 50%;
  filter: blur(100px);
  animation: float 20s ease-in-out infinite;
}

.blob-orange {
  background: #ef5050;
  opacity: 0.08;
  top: -10%;
  left: 10%;
  animation-delay: 0s;
}

.blob-purple {
  background: #fda4af;
  opacity: 0.1;
  bottom: -10%;
  right: 10%;
  animation-delay: -5s;
}

/* ── Login Card ────────────────────────────────────────── */
.login-card {
  width: 100%;
  max-width: 440px;
  background: rgba(255, 255, 255, 0.75);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border: 1px solid rgba(0, 0, 0, 0.06);
  border-radius: 10px;
  box-shadow: 0 20px 40px -15px rgba(15, 23, 42, 0.08);
  padding: 2.5rem;
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
  position: relative;
  z-index: 10;
  box-sizing: border-box;
}

.brand-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
}

.brand-logo {
  width: 80%;
  height: 20px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  margin-bottom: 0.75rem;
}

.brand-tagline {
  color: #64748b;
  font-size: 0.85rem;
  margin: 0.25rem 0 0 0;
}

.divider {
  height: 1px;
  background: linear-gradient(90deg, rgba(0,0,0,0), rgba(0,0,0,0.06), rgba(0,0,0,0));
}

.login-heading {
  color: #1e293b;
  font-size: 1.25rem;
  font-weight: 600;
  text-align: center;
  margin: 0 0 0.5rem 0;
}

.login-subtext {
  color: #475569;
  font-size: 0.88rem;
  line-height: 1.5;
  text-align: center;
  margin: 0 0 1.5rem 0;
}

.signin-button-wrapper {
  display: flex;
  justify-content: center;
  margin-bottom: 1.5rem;
}

/* ── Error Banner ──────────────────────────────────────── */
.error-banner {
  background: rgba(239, 68, 68, 0.05);
  border: 1px solid rgba(239, 68, 68, 0.15);
  border-radius: 10px;
  padding: 0.75rem 1rem;
  display: flex;
  gap: 0.75rem;
  align-items: flex-start;
  margin-top: 1rem;
}

.error-icon {
  width: 18px;
  height: 18px;
  color: #ef4444;
  flex-shrink: 0;
  margin-top: 2px;
}

.error-text {
  color: #ef4444;
  font-size: 0.82rem;
  line-height: 1.4;
  text-align: left;
}

.login-footer {
  text-align: center;
  font-size: 0.75rem;
  color: #64748b;
  line-height: 1.4;
}

/* ── Admin Modal Overlay ────────────────────────────────── */
.admin-modal-overlay {
  position: fixed;
  inset: 0;
  width: 100%;
  height: 77vh;
  background: rgba(241, 245, 249, 0.8);
  /* backdrop-filter: blur(10px); */
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000000;
  font-family: 'Inter', system-ui, sans-serif;
  box-sizing: border-box;
}

.admin-card {
  width: 100%;
  max-width: 580px;
  background: #ffffff;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  box-shadow: 0 20px 40px -15px rgba(15, 23, 42, 0.1);
  display: flex;
  flex-direction: column;
  height: 50vh;
  box-sizing: border-box;
}

.admin-header {
  padding: 1rem 1rem;
  border-bottom: 1px solid #e2e8f0;
  display: flex;
  justify-content: space-between;
  align-items: center;
  /* background-color: paleturquoise; */
  height: 44px;
}

.admin-header-title {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.admin-header-title h2 {
  color: #1e293b;
  font-size: .8rem;
  font-weight: 500;
  margin: 0;
}

.admin-icon {
  width: 20px;
  height: 20px;
  color: #ef5050;
}

.close-btn {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 4px;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s ease;
}

.close-btn:hover {
  background: #f1f5f9;
  color: #1e293b;
}

.close-btn svg {
  width: 20px;
  height: 20px;
}

.admin-body {
  padding: 1.5rem;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.success-banner {
  background: rgba(34, 197, 94, 0.05);
  border: 1px solid rgba(34, 197, 94, 0.15);
  border-radius: 8px;
  padding: 0.75rem 1rem;
  display: flex;
  gap: 0.75rem;
  align-items: center;
  color: #166534;
  font-size: 0.8rem;
}

.success-icon {
  width: 18px;
  height: 18px;
  color: #22c55e;
  flex-shrink: 0;
}

.mb-3 {
  margin-bottom: 0.75rem;
}

/* ── Add Email Form ────────────────────────────────────── */
.add-form {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.form-textarea {
  min-height: 30px;
  resize: vertical;
  font-family: inherit;
  line-height: 1.4;
}

.parsed-preview-box {
  background: #f8fafc;
  border: 1px dashed #cbd5e1;
  border-radius: 8px;
  padding: 0.75rem;
  max-height: 120px;
  overflow-y: auto;
}

.preview-title {
  display: block;
  font-size: 0.75rem;
  font-weight: 600;
  color: #475569;
  margin-bottom: 0.5rem;
  text-align: left;
}

.preview-tags {
  display: flex;
  flex-wrap: wrap;
  gap: 0.375rem;
}

.preview-tag {
  background: #e2e8f0;
  color: #334155;
  font-size: 0.72rem;
  padding: 0.15rem 0.5rem;
  border-radius: 4px;
  border: 1px solid #cbd5e1;
  word-break: break-all;
}

.form-input {
  flex: 1;
  background: #b6b6b620;
  border: 1px solid #383838;
  border-radius: 8px;
  padding: 10px;
  color: #1e293b;
  font-size: 0.7rem;
  outline: none;
  transition: border-color 0.2s ease;
}

.form-input:focus {
  border-color: #ef5050;
}

.submit-btn {
  background: #ef5050;
  color: white;
  border: none;
  border-radius: 5px;
  padding: 5px 10px;
  font-weight: 600;
  font-size: 0.7rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 30%;
  transition: background 0.2s ease;
}

.submit-btn:hover:not(:disabled) {
  background: #db3b3b;
}

.submit-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.btn-spinner {
  width: 16px;
  height: 16px;
  border: 2px solid rgba(255,255,255,0.3);
  border-left-color: white;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

/* ── Whitelisted Email List ────────────────────────────── */
.whitelisted-heading {
  font-size: 0.6rem;
  text-transform: uppercase;
  color: #64748b;
  letter-spacing: 0.05em;
  font-weight: 500;
  border-bottom: 1px solid #e2e8f0;
  padding-bottom: 0.5rem;
  margin-top: 0.5rem;
}

.emails-list-container {
  max-height: 250px;
  overflow-y: auto;
  border: 1px solid #e2e8f0;
  border-radius: 4px;
  background: #f8fafc;
}

.no-emails {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.88rem;
}

.emails-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.email-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 5px;
  border-bottom: 1px solid #e2e8f0;
  transition: background 0.15s ease;
}

.email-item:last-child {
  border-bottom: none;
}

.email-item:hover {
  background: #f1f5f9;
}

.email-details {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.email-text {
  color: #334155;
  font-size: 0.7rem;
}

.badge-admin {
  background: rgba(239, 80, 80, 0.1);
  color: #ef5050;
  font-size: 0.6rem;
  font-weight: 300;
  padding: 2px 8px;
  border: 1px solid rgba(239, 80, 80, 0.2);
  border-radius: 10px;
}

.delete-btn {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 5px;
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s ease;
  width: 23px;
}

.delete-btn:hover:not(:disabled) {
  background: rgba(239, 68, 68, 0.05);
  color: #ef4444;
}

.delete-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.trash-icon {
  width: 16px;
  height: 16px;
}

/* ── Animations ────────────────────────────────────────── */
@keyframes spin {
  to { transform: rotate(360deg); }
}

@keyframes float {
  0% { transform: translateY(0px) rotate(0deg); }
  50% { transform: translateY(-20px) rotate(180deg); }
  100% { transform: translateY(0px) rotate(360deg); }
}

/* Vue Transitions */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

.slide-up-enter-active {
  transition: all 0.3s ease-out;
}

.slide-up-enter-from {
  opacity: 0;
  transform: translateY(10px);
}
</style>
