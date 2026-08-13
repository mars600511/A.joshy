'use strict';

/* ══════════════════════════════════════════
   WEDDING WEBSITE — JAVASCRIPT
   Anjana & Akshay
   Muhoortham: Sunday, 23 August 2026, 10:30 AM IST
══════════════════════════════════════════ */

// ── Wedding date target ────────────────────
const WEDDING_DATE = new Date('2026-08-23T10:30:00+05:30');

// ── DOM refs ───────────────────────────────
const splash        = document.getElementById('splash');
const details       = document.getElementById('details');
const swipeTrack    = document.getElementById('swipeTrack');
const swipeThumb    = document.getElementById('swipeThumb');
const swipeLabel    = document.getElementById('swipeLabel');
const swipeFill     = document.getElementById('swipeFill');
const bgMusic       = document.getElementById('bgMusic');
const musicPill     = document.getElementById('musicPill');
const musicPillDet  = document.getElementById('musicPillDetails');

// ── State ──────────────────────────────────
let musicStarted = false;  // has audio.play() ever succeeded?
let musicMuted   = false;  // is it currently muted?

/* ════════════════════════════════════════════
   AUDIO UNLOCK — iOS Safari requires a play()
   call during a user gesture before any later
   play() will succeed. We trigger a silent
   play/pause on the very first touch so that
   by the time the swipe completes, the audio
   context is already unlocked.
════════════════════════════════════════════ */
(function unlockAudioOnFirstTouch() {
  if (!bgMusic) return;
  function unlock() {
    bgMusic.muted = true;
    bgMusic.play().then(() => {
      bgMusic.pause();
      bgMusic.currentTime = 0;
      bgMusic.muted = false;
    }).catch(() => {});
    document.removeEventListener('touchstart', unlock, true);
    document.removeEventListener('mousedown',  unlock, true);
  }
  document.addEventListener('touchstart', unlock, { capture: true, once: true, passive: true });
  document.addEventListener('mousedown',  unlock, { capture: true, once: true });
})();

/* ════════════════════════════════════════════
   COUNTDOWN TIMER
════════════════════════════════════════════ */
const cdDays  = document.getElementById('cd-days');
const cdHours = document.getElementById('cd-hours');
const cdMins  = document.getElementById('cd-mins');
const cdSecs  = document.getElementById('cd-secs');

function pad(n) { return String(Math.max(0, n)).padStart(2, '0'); }

function animateDigit(el, newVal) {
  if (el.textContent === newVal) return;
  el.classList.remove('flip');
  // Force reflow
  void el.offsetWidth;
  el.textContent = newVal;
  el.classList.add('flip');
}

function updateCountdown() {
  const now  = Date.now();
  const diff = WEDDING_DATE.getTime() - now;

  if (diff <= 0) {
    animateDigit(cdDays,  '00');
    animateDigit(cdHours, '00');
    animateDigit(cdMins,  '00');
    animateDigit(cdSecs,  '00');
    return;
  }

  const totalSecs = Math.floor(diff / 1000);
  const days  = Math.floor(totalSecs / 86400);
  const hours = Math.floor((totalSecs % 86400) / 3600);
  const mins  = Math.floor((totalSecs % 3600) / 60);
  const secs  = totalSecs % 60;

  animateDigit(cdDays,  pad(days));
  animateDigit(cdHours, pad(hours));
  animateDigit(cdMins,  pad(mins));
  animateDigit(cdSecs,  pad(secs));
}

updateCountdown();
setInterval(updateCountdown, 1000);

/* ════════════════════════════════════════════
   SWIPE-TO-ATTEND
════════════════════════════════════════════ */
(function initSwipe() {
  let isDragging = false;
  let startX     = 0;
  let currentX   = 0;
  let trackW, thumbW, maxTravel;

  function getGeometry() {
    trackW    = swipeTrack.offsetWidth;
    thumbW    = swipeThumb.offsetWidth;
    maxTravel = trackW - thumbW - 16; // 8px padding each side
  }

  function setThumbX(x) {
    x = Math.max(0, Math.min(x, maxTravel));
    currentX = x;
    const progress = x / maxTravel;

    swipeThumb.style.transform = `translateY(-50%) translateX(${x}px)`;
    swipeLabel.style.opacity = String(Math.max(0, 1 - progress * 2));
    swipeFill.style.width = `${progress * 100}%`;
    swipeTrack.style.border = `1px solid rgba(255,255,255,${0.25 + progress * 0.5})`;
  }

  function onDragStart(clientX) {
    getGeometry();
    isDragging = true;
    startX     = clientX - currentX;
    swipeThumb.style.transition = 'none';
    swipeFill.style.transition  = 'none';
    swipeTrack.style.cursor     = 'grabbing';
  }

  function onDragMove(clientX) {
    if (!isDragging) return;
    setThumbX(clientX - startX);
  }

  function onDragEnd() {
    if (!isDragging) return;
    isDragging = false;
    swipeTrack.style.cursor = '';

    if (currentX / maxTravel >= 0.78) {
      completeSwipe();
    } else {
      snapBack();
    }
  }

  function snapBack() {
    swipeThumb.style.transition = 'transform 0.45s cubic-bezier(0.32,0.72,0,1)';
    swipeFill.style.transition  = 'width 0.45s cubic-bezier(0.32,0.72,0,1)';
    swipeLabel.style.transition = 'opacity 0.3s';
    setThumbX(0);
    currentX = 0;
    swipeLabel.style.opacity = '1';
  }

  function completeSwipe() {
    getGeometry();
    swipeThumb.style.transition = 'transform 0.3s cubic-bezier(0.32,0.72,0,1)';
    swipeFill.style.transition  = 'width 0.3s ease';
    setThumbX(maxTravel);

    swipeTrack.classList.add('done');
    swipeLabel.style.opacity    = '0';
    swipeLabel.style.paddingLeft = '0';

    setTimeout(() => {
      swipeLabel.textContent   = 'See you there! ✓';
      swipeLabel.style.opacity = '1';
    }, 250);

    // ── Start music SYNCHRONOUSLY during user gesture ──────────
    // bgMusic.play() must be called within the touchend/mouseup
    // event handler — any setTimeout breaks the autoplay policy.
    if (bgMusic && !musicStarted) {
      bgMusic.volume = 0;
      bgMusic.play().then(() => {
        musicStarted = true;
        musicMuted   = false;
        updateMusicUI();
        fadeVolume(0, 0.55, 2000);
      }).catch(() => {
        // Autoplay blocked — user can tap the music pill to start
      });
    }

    // Page reveal can safely be deferred (it's UI only)
    setTimeout(revealDetails, 700);
  }

  // ── Touch events ───────────────────────────
  swipeThumb.addEventListener('touchstart', e => {
    e.preventDefault();
    onDragStart(e.touches[0].clientX);
  }, { passive: false });

  document.addEventListener('touchmove', e => {
    if (isDragging) {
      e.preventDefault();
      onDragMove(e.touches[0].clientX);
    }
  }, { passive: false });

  document.addEventListener('touchend', () => onDragEnd());

  // ── Mouse events (desktop) ─────────────────
  swipeThumb.addEventListener('mousedown', e => {
    e.preventDefault();
    onDragStart(e.clientX);
  });
  document.addEventListener('mousemove', e => {
    if (isDragging) onDragMove(e.clientX);
  });
  document.addEventListener('mouseup', () => onDragEnd());

  // Resize
  window.addEventListener('resize', () => {
    if (!swipeTrack.classList.contains('done')) { currentX = 0; }
  });
})();

/* ════════════════════════════════════════════
   PAGE TRANSITION — REVEAL DETAILS
════════════════════════════════════════════ */
function revealDetails() {
  details.classList.add('revealed');
  details.removeAttribute('aria-hidden');

  splash.classList.add('exit');
  setTimeout(() => { splash.style.visibility = 'hidden'; }, 900);

}

/* ════════════════════════════════════════════
   BACKGROUND MUSIC
════════════════════════════════════════════ */
function fadeVolume(from, to, durationMs) {
  const steps    = 40;
  const interval = durationMs / steps;
  const delta    = (to - from) / steps;
  let   current  = from;
  const timer    = setInterval(() => {
    current = Math.max(0, Math.min(1, current + delta));
    bgMusic.volume = current;
    if ((delta > 0 && current >= to) || (delta < 0 && current <= to)) {
      clearInterval(timer);
    }
  }, interval);
}

function toggleMusic() {
  if (!bgMusic) return;

  if (!musicStarted) {
    // Audio hasn't started yet (swipe not done / autoplay blocked)
    // This click IS a user gesture so play() will work here
    bgMusic.volume = 0;
    bgMusic.play().then(() => {
      musicStarted = true;
      musicMuted   = false;
      updateMusicUI();
      fadeVolume(0, 0.55, 1000);
    }).catch(() => {});
    return;
  }

  // Toggle mute / unmute (don't pause — keep buffer position)
  musicMuted    = !musicMuted;
  bgMusic.muted = musicMuted;
  updateMusicUI();
}

function updateMusicUI() {
  const isAudible = musicStarted && !musicMuted;
  const pills = [musicPill, musicPillDet].filter(Boolean);
  pills.forEach(p => {
    if (isAudible) {
      p.classList.add('playing');
      p.setAttribute('aria-label', 'Mute music');
    } else {
      p.classList.remove('playing');
      p.setAttribute('aria-label', 'Play Nikkah Nasheed');
    }
  });
}

if (musicPill)    musicPill.addEventListener('click', toggleMusic);
if (musicPillDet) musicPillDet.addEventListener('click', toggleMusic);

/* ════════════════════════════════════════════
   DOWNLOAD CARD BUTTON
════════════════════════════════════════════ */
async function downloadWeddingCard(btn, cardUrl, filename) {
  const CARD_URL = cardUrl || 'assets/IMG-20260811-WA0089 (1).jpg';
  const FILENAME = filename || 'Anjana-Akshay-WeddingCard.jpg';

  const origText = btn.textContent;
  btn.textContent = 'Downloading…';
  btn.disabled    = true;

  try {
    const res  = await fetch(CARD_URL);
    if (!res.ok) throw new Error('Card image not found');
    const blob = await res.blob();
    const file = new File([blob], FILENAME, { type: blob.type || 'image/png' });

    // Mobile: native share sheet (WhatsApp status, etc.)
    if (navigator.canShare && navigator.canShare({ files: [file] })) {
      await navigator.share({
        files: [file],
        title: 'Anjana & Akshay — Wedding Card',
        text:  'You\'re invited! ✨'
      });
    } else {
      // Desktop / fallback: trigger download
      const url = URL.createObjectURL(blob);
      const a   = document.createElement('a');
      a.href     = url;
      a.download = FILENAME;
      a.click();
      setTimeout(() => URL.revokeObjectURL(url), 60000);
    }
  } catch {
    // Silently fail — card image may not be uploaded yet
  } finally {
    btn.textContent = origText;
    btn.disabled    = false;
  }
}

document.querySelectorAll('.download-btn:not(.disabled)').forEach(btn => {
  btn.addEventListener('click', () => downloadWeddingCard(btn));
});

/* ── Download second card ────────────────── */
document.querySelectorAll('.download-btn-2:not(.disabled)').forEach(btn => {
  btn.addEventListener('click', () => downloadWeddingCard(btn, 'assets/IMG-20260811-WA0090.jpg', 'Anjana-Akshay-WeddingCard-2.jpg'));
});


/* ════════════════════════════════════════════
   HAPTIC FEEDBACK (PWA / Safari)
════════════════════════════════════════════ */
function vibrate(pattern) {
  if ('vibrate' in navigator) navigator.vibrate(pattern);
}

swipeThumb.addEventListener('touchstart', () => vibrate(10), { passive: true });

/* ════════════════════════════════════════════
   RSVP TRACKING
════════════════════════════════════════════ */
const GOOGLE_APPS_SCRIPT_URL = 'https://script.google.com/macros/d/AKfycby6xBILS6Ejri5iqkPtWTtcHJKhR1JkMpDfZg2lV02pD8vS28EdeLxj5YwzXxGxQ3kW/usercontent';
const rsvpModal = document.getElementById('rsvpModal');
const rsvpForm = document.getElementById('rsvpForm');
const rsvpClose = document.getElementById('rsvpClose');
const countNumber = document.getElementById('countNumber');

// Load attendee count from localStorage
function loadAttendeeCount() {
  const count = localStorage.getItem('rsvpCount') || '0';
  countNumber.textContent = count;
  return parseInt(count);
}

// Increment attendee count
function incrementAttendeeCount() {
  let count = parseInt(localStorage.getItem('rsvpCount') || '0');
  count++;
  localStorage.setItem('rsvpCount', count);
  countNumber.textContent = count;
}

// Show RSVP modal
function showRsvpModal() {
  rsvpModal.classList.add('open');
  rsvpModal.removeAttribute('aria-hidden');
}

// Hide RSVP modal
function hideRsvpModal() {
  rsvpModal.classList.remove('open');
  rsvpModal.setAttribute('aria-hidden', 'true');
}

// Handle RSVP form submission
rsvpForm.addEventListener('submit', async (e) => {
  e.preventDefault();
  
  const name = document.getElementById('guestName').value.trim();
  const phone = document.getElementById('guestPhone').value.trim();
  
  if (!name || !phone) return;
  
  const submitBtn = rsvpForm.querySelector('.rsvp-submit');
  const origText = submitBtn.textContent;
  submitBtn.disabled = true;
  submitBtn.textContent = 'Saving...';
  
  try {
    // Send to Google Sheets
    const response = await fetch(GOOGLE_APPS_SCRIPT_URL, {
      method: 'POST',
      mode: 'no-cors',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        name: name,
        phone: phone,
        timestamp: new Date().toLocaleString()
      })
    });
    
    // Increment counter
    incrementAttendeeCount();
    
    // Reset form
    rsvpForm.reset();
    
    // Show success message
    submitBtn.textContent = '✓ RSVP Confirmed!';
    setTimeout(() => {
      hideRsvpModal();
      submitBtn.textContent = origText;
      submitBtn.disabled = false;
    }, 1500);
  } catch (error) {
    submitBtn.textContent = origText;
    submitBtn.disabled = false;
  }
});

// Close modal handlers
rsvpClose.addEventListener('click', hideRsvpModal);
rsvpModal.addEventListener('click', (e) => {
  if (e.target === rsvpModal || e.target === document.querySelector('.rsvp-overlay')) {
    hideRsvpModal();
  }
});

// Load initial count
loadAttendeeCount();

/* ════════════════════════════════════════════
   MODIFY SWIPE TO SHOW RSVP MODAL
════════════════════════════════════════════ */
// Store original completeSwipe function
const originalCompleteSwipe = window.completeSwipeFunc;

// We'll modify the completeSwipe within the initSwipe IIFE by intercepting the details reveal
const originalRevealDetails = revealDetails;
function revealDetailsWithRsvp() {
  originalRevealDetails();
  // Show RSVP modal after a brief delay
  setTimeout(showRsvpModal, 800);
}
revealDetails = revealDetailsWithRsvp;



/* ════════════════════════════════════════════
   TRANSPORT ACCORDION
════════════════════════════════════════════ */
(function initAccordion() {
  const items = document.querySelectorAll('.accordion-item');
  items.forEach(item => {
    const header = item.querySelector('.accordion-header');
    if (!header) return;
    header.addEventListener('click', () => {
      const isOpen = item.classList.contains('open');
      // Close all
      items.forEach(i => {
        i.classList.remove('open');
        i.querySelector('.accordion-header').setAttribute('aria-expanded', 'false');
      });
      // Open the clicked one if it was closed
      if (!isOpen) {
        item.classList.add('open');
        header.setAttribute('aria-expanded', 'true');
      }
    });
  });
})();
