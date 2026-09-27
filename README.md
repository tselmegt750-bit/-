<!DOCTYPE html>
<html lang="mn" data-theme="dark">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Тан Эрдэм — Эмийн Жорын Тоглоом</title>
<link href="https://fonts.googleapis.com/css2?family=Dela+Gothic+One&family=Nunito:wght@400;700;900&display=swap" rel="stylesheet">

<!-- ═══════════════════════════════════════════════
  FIREBASE ТОХИРГОО — https://console.firebase.google.com
  1. Шинэ Project үүсгэх
  2. Authentication → Google нэвтрэлт идэвхжүүлэх
  3. Storage идэвхжүүлэх
  4. Доорх config-г өөрийнхөөр солих
═══════════════════════════════════════════════ -->
<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
import { getAuth, signInWithPopup, GoogleAuthProvider, signOut, onAuthStateChanged }
  from "https://www.gstatic.com/firebasejs/10.12.0/firebase-auth.js";
import { getFirestore, doc, getDoc, setDoc, collection, addDoc, serverTimestamp, query, orderBy, limit, getDocs }
  from "https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js";
import { getStorage, ref, uploadBytes, getDownloadURL }
  from "https://www.gstatic.com/firebasejs/10.12.0/firebase-storage.js";

// ╔══════════════════════════════════════╗
// ║  ЭНЭХҮҮ ХЭСГИЙГ ӨӨРИЙНХӨӨР СОЛИНО ║
const firebaseConfig = {
  apiKey:            "ЭНД_ӨӨРИЙН_API_KEY",
  authDomain:        "ЭНД_ӨӨРИЙН_PROJECT.firebaseapp.com",
  projectId:         "ЭНД_ӨӨРИЙН_PROJECT_ID",
  storageBucket:     "ЭНД_ӨӨРИЙН_PROJECT.appspot.com",
  messagingSenderId: "ЭНД_ӨӨРИЙН_SENDER_ID",
  appId:             "ЭНД_ӨӨРИЙН_APP_ID"
};
// ╚══════════════════════════════════════╝

const app  = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db   = getFirestore(app);
const storage = getStorage(app);
const provider = new GoogleAuthProvider();

const FREE_LIMIT = 20;
const MONTHLY_FEE = 5000;
const KHAAN_ACCOUNT = "5022*****";  // ← Өөрийн Хаан банкны дансаа оруулна

let currentUser = null;
let isPremium = false;
let premiumExpiry = null;

// ── AUTH STATE ──
onAuthStateChanged(auth, async (user) => {
  if (user) {
    currentUser = user;
    document.getElementById("authScreen").classList.add("hidden");
    document.getElementById("mainGame").style.display = "block";
    document.getElementById("userAvatar").src = user.photoURL || "";
    document.getElementById("userName").textContent = user.displayName || user.email;
    window.playerName = user.displayName || "Тоглогч";
    await checkPremium(user.uid);
    initGame();
  } else {
    currentUser = null;
    document.getElementById("authScreen").classList.remove("hidden");
    document.getElementById("mainGame").style.display = "none";
  }
});

// ── PREMIUM ШАЛГАХ ──
async function checkPremium(uid) {
  const snap = await getDoc(doc(db, "users", uid));
  if (snap.exists()) {
    const data = snap.data();
    const now = Date.now();
    if (data.premiumUntil && data.premiumUntil.toMillis() > now) {
      isPremium = true;
      premiumExpiry = data.premiumUntil.toDate();
      document.getElementById("premiumBadge").style.display = "inline-flex";
      document.getElementById("premiumBadge").textContent =
        "⭐ Premium — " + premiumExpiry.toLocaleDateString("mn-MN");
    } else {
      isPremium = false;
      document.getElementById("premiumBadge").style.display = "none";
    }
  }
  updatePremiumUI();
}

function updatePremiumUI() {
  const banner = document.getElementById("premiumBanner");
  if (isPremium) { banner.style.display = "none"; }
  else { banner.style.display = "flex"; }
}

// ── GOOGLE НЭВТРЭХ ──
window.signInGoogle = async () => {
  try {
    document.getElementById("loginBtn").disabled = true;
    document.getElementById("loginBtn").textContent = "Нэвтэрч байна...";
    await signInWithPopup(auth, provider);
  } catch(e) {
    alert("Нэвтрэхэд алдаа гарлаа: " + e.message);
    document.getElementById("loginBtn").disabled = false;
    document.getElementById("loginBtn").textContent = "🔑 Google-ээр нэвтрэх";
  }
};

// ── ГАРАХ ──
window.doSignOut = async () => {
  await signOut(auth);
};

// ── КАРТЫН ХЯНАЛТ (FREE LIMIT) ──
window.checkAccessAndNext = (idx) => {
  if (!isPremium && idx >= FREE_LIMIT) {
    document.getElementById("paywall").classList.add("show");
    return false;
  }
  return true;
};

// ── ЗУРАГ UPLOAD & ТӨЛБӨР ХҮСЭЛТ ──
window.submitPayment = async () => {
  const fileInput = document.getElementById("receiptFile");
  const file = fileInput.files[0];
  if (!file) { alert("Дэлгэцийн зургаа оруулна уу!"); return; }

  const btn = document.getElementById("submitPayBtn");
  btn.disabled = true; btn.textContent = "Илгээж байна...";

  try {
    const storageRef = ref(storage, "receipts/" + currentUser.uid + "_" + Date.now());
    await uploadBytes(storageRef, file);
    const url = await getDownloadURL(storageRef);

    await addDoc(collection(db, "payments"), {
      uid:       currentUser.uid,
      email:     currentUser.email,
      name:      currentUser.displayName,
      photoUrl:  url,
      status:    "pending",
      amount:    MONTHLY_FEE,
      createdAt: serverTimestamp()
    });

    document.getElementById("paymentSuccess").style.display = "block";
    document.getElementById("paymentForm").style.display = "none";
    btn.disabled = false; btn.textContent = "✅ Илгээх";
  } catch(e) {
    alert("Алдаа: " + e.message);
    btn.disabled = false; btn.textContent = "✅ Илгээх";
  }
};

// ── LEADERBOARD FIRESTORE-ООС УНШИХ ──
window.loadFirestoreLB = async () => {
  try {
    const q = query(collection(db, "leaderboard"), orderBy("score","desc"), limit(20));
    const snap = await getDocs(q);
    const medals = ["🥇","🥈","🥉"];
    const today = new Date().toLocaleDateString("mn-MN");
    let html = "";
    let i = 0;
    snap.forEach(d => {
      const r = d.data();
      if (r.date === today) {
        html += `<div class="lb-row">
          <div class="lb-rank">${medals[i] || (i+1)}</div>
          <img src="${r.photoURL || ""}" class="lb-avatar" onerror="this.style.display='none'">
          <div><div class="lb-name">${r.name}</div><div class="lb-date">${r.date}</div></div>
          <div class="lb-score">${r.score} оноо</div>
        </div>`;
        i++;
      }
    });
    document.getElementById("lbList").innerHTML = html || "<div class='lb-empty'>Өнөөдрийн бичлэг байхгүй</div>";
  } catch(e) { console.log("LB error:", e); }
};

// ── ОНОО FIRESTORE-Д ХАДГАЛАХ ──
window.saveScoreFirestore = async (pts) => {
  if (!currentUser) return;
  try {
    const today = new Date().toLocaleDateString("mn-MN");
    const docRef = doc(db, "leaderboard", currentUser.uid + "_" + today);
    const snap = await getDoc(docRef);
    if (!snap.exists() || snap.data().score < pts) {
      await setDoc(docRef, {
        uid:      currentUser.uid,
        name:     currentUser.displayName,
        email:    currentUser.email,
        photoURL: currentUser.photoURL,
        score:    pts,
        date:     today,
        updatedAt: serverTimestamp()
      });
    }
  } catch(e) { console.log("Score save error:", e); }
};

window.__firebase = { checkPremium, loadFirestoreLB, saveScoreFirestore };
</script>

<style>
:root{--gold:#D4A017;--deep:#1a0a2e;--green:#2d6a4f;--lg:#52b788;--red:#c1121f;--bg:#0d1b2a;--surface:rgba(255,255,255,0.07);--border:rgba(255,255,255,0.13);--text:#fff;--text2:rgba(255,255,255,.55);--card-front:linear-gradient(135deg,#1e3a5f,#0d2137);--card-back:linear-gradient(135deg,#1a3d2b,#0d2119)}
[data-theme="light"]{--bg:#f0f4f8;--surface:rgba(0,0,0,.06);--border:rgba(0,0,0,.12);--text:#1a2635;--text2:rgba(0,0,0,.5);--card-front:linear-gradient(135deg,#c8dff7,#e8f4ff);--card-back:linear-gradient(135deg,#c8f0da,#e8fff2)}
*{box-sizing:border-box;margin:0;padding:0}
body{font-family:"Nunito",sans-serif;background:var(--bg);min-height:100vh;overflow-x:hidden;color:var(--text);transition:background .3s,color .3s}
body::before{content:"";position:fixed;inset:0;background:radial-gradient(ellipse at 20% 20%,rgba(212,160,23,.12) 0%,transparent 50%),radial-gradient(ellipse at 80% 80%,rgba(45,106,79,.15) 0%,transparent 50%);z-index:0;pointer-events:none}

/* ── НЭВТРЭХ ДЭЛГЭЦ ── */
.auth-screen{position:fixed;inset:0;background:var(--bg);z-index:500;display:flex;flex-direction:column;align-items:center;justify-content:center;padding:30px 20px;text-align:center}
.auth-screen.hidden{display:none}
.auth-logo{font-size:72px;margin-bottom:16px;animation:bounce .8s ease infinite alternate}
@keyframes bounce{from{transform:translateY(0)}to{transform:translateY(-12px)}}
.auth-title{font-family:"Dela Gothic One",cursive;font-size:28px;background:linear-gradient(to right,var(--gold),var(--lg));-webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text;margin-bottom:8px}
.auth-sub{color:var(--text2);font-size:14px;margin-bottom:32px;line-height:1.6}
.btn-google{display:flex;align-items:center;gap:12px;padding:16px 32px;background:#fff;color:#1f2937;border:none;border-radius:50px;font-family:"Nunito",sans-serif;font-weight:900;font-size:16px;cursor:pointer;transition:all .2s;box-shadow:0 8px 25px rgba(0,0,0,.2);margin-bottom:12px}
.btn-google:hover{transform:scale(1.04);box-shadow:0 12px 30px rgba(0,0,0,.25)}
.btn-google img{width:24px;height:24px}
.auth-note{font-size:12px;color:var(--text2);max-width:320px;line-height:1.6}
.free-badge{display:inline-block;background:linear-gradient(135deg,var(--green),var(--lg));color:#fff;padding:6px 16px;border-radius:20px;font-size:12px;font-weight:700;margin-bottom:20px}

/* ── HEADER ── */
.particles{position:fixed;inset:0;z-index:0;pointer-events:none}
.particle{position:absolute;border-radius:50%;animation:floatUp linear infinite;opacity:.35}
@keyframes floatUp{0%{transform:translateY(100vh) rotate(0deg);opacity:0}10%{opacity:.35}90%{opacity:.35}100%{transform:translateY(-10vh) rotate(720deg);opacity:0}}
header{position:relative;z-index:10;text-align:center;padding:16px 20px 8px}
.top-bar{display:flex;justify-content:space-between;align-items:center;max-width:700px;margin:0 auto 10px;padding:0 4px;gap:8px}
.user-info{display:flex;align-items:center;gap:8px;background:var(--surface);border:1px solid var(--border);border-radius:20px;padding:6px 12px;flex:1;min-width:0}
.user-avatar{width:28px;height:28px;border-radius:50%;object-fit:cover;border:2px solid var(--gold);flex-shrink:0}
.user-name{font-size:12px;font-weight:700;color:var(--gold);white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
.premium-badge{font-size:10px;color:var(--lg);font-weight:700;display:none}
.top-actions{display:flex;gap:6px;flex-shrink:0}
.icon-btn{background:var(--surface);border:1px solid var(--border);border-radius:12px;padding:7px 10px;font-size:14px;cursor:pointer;transition:all .2s;color:var(--text)}
.icon-btn:hover{background:var(--border)}
.logo-badge{display:inline-block;background:linear-gradient(135deg,var(--gold),#a0722a);padding:5px 18px;border-radius:20px;font-size:11px;font-weight:900;letter-spacing:3px;text-transform:uppercase;margin-bottom:8px;color:#1a0a2e}
h1{font-family:"Dela Gothic One",cursive;font-size:clamp(20px,5vw,40px);background:linear-gradient(to right,var(--gold),#fff,var(--lg));-webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text;line-height:1.1;margin-bottom:4px}
[data-theme="light"] h1{background:linear-gradient(to right,#b8860b,#2d6a4f);-webkit-background-clip:text;background-clip:text}
.subtitle{color:var(--text2);font-size:12px;letter-spacing:1px}

/* ── PREMIUM BANNER ── */
.premium-banner{display:none;background:linear-gradient(135deg,var(--gold),#a0722a);margin:8px 16px;border-radius:16px;padding:12px 16px;align-items:center;gap:12px;position:relative;z-index:10;max-width:700px;margin-left:auto;margin-right:auto}
.premium-banner-text{flex:1;font-size:13px;font-weight:700;color:#1a0a2e;line-height:1.4}
.btn-upgrade{background:#1a0a2e;color:var(--gold);border:none;border-radius:12px;padding:8px 16px;font-family:"Nunito",sans-serif;font-weight:900;font-size:12px;cursor:pointer;white-space:nowrap}

/* ── SCORE, PROGRESS, TIMER (v2-с хуулсан) ── */
.score-bar{position:relative;z-index:10;display:flex;justify-content:center;gap:12px;padding:10px 20px}
.score-item{background:var(--surface);border:1px solid var(--border);border-radius:16px;padding:7px 14px;text-align:center}
.score-item .val{font-size:24px;font-weight:900;line-height:1}
.score-item .lbl{font-size:10px;color:var(--text2);text-transform:uppercase;letter-spacing:1px}
.score-item.gold .val{color:var(--gold)}.score-item.green .val{color:var(--lg)}.score-item.red .val{color:#ff6b6b}
.progress-wrap{position:relative;z-index:10;padding:0 20px 8px;max-width:700px;margin:0 auto}
.progress-track{background:var(--surface);border-radius:10px;height:10px;overflow:hidden;border:1px solid var(--border)}
.progress-fill{height:100%;background:linear-gradient(to right,var(--green),var(--lg),var(--gold));border-radius:10px;transition:width .6s cubic-bezier(.34,1.56,.64,1);position:relative}
.progress-fill::after{content:"";position:absolute;right:0;top:0;bottom:0;width:6px;background:white;border-radius:3px;opacity:.6;animation:pulse 1s ease infinite alternate}
@keyframes pulse{from{opacity:.3}to{opacity:.9}}
.progress-label{display:flex;justify-content:space-between;font-size:11px;color:var(--text2);margin-top:4px}
.timer-bar-wrap{max-width:700px;margin:0 auto;padding:0 20px 6px;position:relative;z-index:10}
.timer-bar-track{background:var(--surface);border-radius:6px;height:6px;overflow:hidden}
.timer-bar-fill{height:100%;width:100%;border-radius:6px;background:var(--lg);transition:width .1s linear,background .3s}
.timer-label{font-size:12px;font-weight:900;color:var(--lg);text-align:right;margin-top:2px}
.dots-row{display:flex;gap:4px;justify-content:center;flex-wrap:wrap;margin-bottom:12px;position:relative;z-index:10;padding:0 20px}
.mini-dot{width:8px;height:8px;border-radius:50%;transition:all .3s}
.mini-dot.known{background:var(--lg)}.mini-dot.unknown{background:var(--red);opacity:.7}
.mini-dot.current{background:var(--gold);transform:scale(1.6)}.mini-dot.unseen{background:var(--surface);border:1px solid var(--border)}
.mini-dot.locked{background:rgba(255,255,255,0.1);border:1px dashed rgba(255,255,255,0.2)}

/* ── CARD AREA ── */
.card-area{position:relative;z-index:10;max-width:700px;margin:0 auto;padding:0 16px 20px}
.mode-tabs{display:flex;gap:6px;margin-bottom:12px;background:var(--surface);border-radius:16px;padding:5px;border:1px solid var(--border)}
.mode-tab{flex:1;padding:9px;border:none;border-radius:12px;font-family:"Nunito",sans-serif;font-weight:700;font-size:12px;cursor:pointer;transition:all .3s;background:transparent;color:var(--text2)}
.mode-tab.active{background:linear-gradient(135deg,var(--green),var(--lg));color:white;box-shadow:0 4px 15px rgba(82,183,136,.35)}
.flashcard-container{margin-bottom:12px;cursor:pointer}
.card-face{border-radius:22px;padding:24px;display:flex;flex-direction:column;justify-content:center;align-items:center;text-align:center;border:1px solid var(--border);width:100%;min-height:200px;transition:opacity .25s}
.card-front{background:var(--card-front);box-shadow:0 16px 50px rgba(0,0,0,.4);display:flex}
.card-back{background:var(--card-back);box-shadow:0 16px 50px rgba(0,0,0,.4);display:none}
.flashcard.flipped .card-front{display:none}.flashcard.flipped .card-back{display:flex}
.card-deco{position:absolute;font-size:52px;opacity:.05;pointer-events:none}
.card-deco-1{top:10px;right:14px}.card-deco-2{bottom:10px;left:14px}
.card-number{font-size:10px;font-weight:900;letter-spacing:3px;text-transform:uppercase;color:var(--gold);margin-bottom:8px;opacity:.85}
.card-title{font-family:"Dela Gothic One",cursive;font-size:clamp(17px,4vw,28px);color:var(--text);margin-bottom:5px;line-height:1.2}
.card-type{font-size:12px;color:var(--lg);font-weight:700;margin-bottom:9px}
.card-ingredients{font-size:12px;color:var(--text2);font-style:italic;line-height:1.6;word-break:break-word}
.card-title-small{font-family:"Dela Gothic One",cursive;font-size:14px;color:var(--lg);margin-bottom:9px}
.card-description{font-size:13px;color:var(--text);line-height:1.8;width:100%;word-break:break-word;text-align:left}
.flip-hint{font-size:11px;color:var(--text2);margin-top:12px}
.nav-row{display:flex;align-items:center;justify-content:space-between;gap:10px;margin-bottom:10px}
.btn-nav{padding:10px 16px;border:1px solid var(--border);border-radius:14px;background:var(--surface);color:var(--text);font-family:"Nunito",sans-serif;font-weight:700;font-size:13px;cursor:pointer;transition:all .2s}
.btn-nav:hover{background:var(--border)}
.card-counter{font-weight:900;color:var(--text2);font-size:13px}
.action-buttons{display:grid;grid-template-columns:1fr 1fr;gap:10px;margin-bottom:12px}
.btn{padding:13px;border:none;border-radius:16px;font-family:"Nunito",sans-serif;font-weight:900;font-size:14px;cursor:pointer;transition:all .2s;display:flex;align-items:center;justify-content:center;gap:6px;position:relative;overflow:hidden}
.btn:active{transform:scale(.96)}
.btn-know{background:linear-gradient(135deg,#2d6a4f,#52b788);color:white;box-shadow:0 5px 18px rgba(82,183,136,.35)}
.btn-dontknow{background:linear-gradient(135deg,#7b2d2d,#c1121f);color:white;box-shadow:0 5px 18px rgba(193,18,31,.35)}
.btn-next{grid-column:span 2;background:linear-gradient(135deg,#1e3a5f,#2d5f9e);color:white;box-shadow:0 5px 18px rgba(45,95,158,.35)}
@keyframes glowGreen{0%{box-shadow:0 0 0 0 rgba(82,183,136,0)}50%{box-shadow:0 0 24px 8px rgba(82,183,136,.7)}100%{box-shadow:0 0 0 0 rgba(82,183,136,0)}}
@keyframes shakeRed{0%,100%{transform:translateX(0)}20%{transform:translateX(-10px)}40%{transform:translateX(10px)}60%{transform:translateX(-8px)}80%{transform:translateX(8px)}}
.correct-anim{animation:glowGreen .6s ease}.wrong-anim{animation:shakeRed .5s ease}
.quiz-question-box{background:var(--surface);border:1px solid var(--border);border-radius:18px;padding:16px;margin-bottom:12px;font-size:14px;font-weight:700;color:var(--text);line-height:1.6}
.quiz-options{display:grid;grid-template-columns:1fr 1fr;gap:10px;margin-bottom:12px}
.quiz-option{padding:13px 10px;border:2px solid var(--border);border-radius:16px;background:var(--surface);color:var(--text);font-family:"Nunito",sans-serif;font-weight:700;font-size:12px;cursor:pointer;transition:all .25s;text-align:center;line-height:1.4}
.quiz-option:hover{border-color:var(--gold);background:rgba(212,160,23,.1);transform:translateY(-2px)}
.quiz-option.correct{border-color:var(--lg);background:rgba(82,183,136,.2);animation:glowGreen .6s ease}
.quiz-option.wrong{border-color:var(--red);background:rgba(193,18,31,.2);animation:shakeRed .5s ease}
.btn-next-full{width:100%;padding:13px;border:none;border-radius:16px;background:linear-gradient(135deg,#1e3a5f,#2d5f9e);color:white;font-family:"Nunito",sans-serif;font-weight:900;font-size:14px;cursor:pointer;transition:all .2s;margin-top:4px}
.spell-question{background:var(--surface);border:1px solid var(--border);border-radius:16px;padding:14px;margin-bottom:10px;font-size:13px;color:var(--text);line-height:1.7;word-break:break-word}
.spell-question strong{color:var(--gold);display:block;font-size:13px;margin-bottom:5px}
#spellInput{width:100%;padding:13px 16px;border-radius:16px;border:2px solid var(--border);background:var(--surface);color:var(--text);font-family:"Nunito",sans-serif;font-size:15px;font-weight:700;outline:none;transition:border-color .2s;margin-bottom:8px}
#spellInput:focus{border-color:var(--lg)}#spellInput::placeholder{color:var(--text2)}
.btn-check{width:100%;background:linear-gradient(135deg,var(--gold),#a0722a);color:#1a0a2e;font-size:14px;margin-bottom:8px;padding:13px;border:none;border-radius:16px;font-family:"Nunito",sans-serif;font-weight:900;cursor:pointer;transition:all .2s}

/* ── PAYWALL MODAL ── */
.paywall-overlay{position:fixed;inset:0;background:rgba(0,0,0,.85);z-index:400;display:none;align-items:center;justify-content:center;padding:20px}
.paywall-overlay.show{display:flex}
.paywall-box{background:var(--bg);border:1px solid var(--border);border-radius:28px;padding:28px 24px;max-width:400px;width:100%;text-align:center;animation:slideUp .4s ease}
@keyframes slideUp{from{transform:translateY(40px);opacity:0}to{transform:translateY(0);opacity:1}}
.paywall-icon{font-size:52px;margin-bottom:12px}
.paywall-title{font-family:"Dela Gothic One",cursive;font-size:22px;color:var(--gold);margin-bottom:8px}
.paywall-desc{color:var(--text2);font-size:13px;line-height:1.7;margin-bottom:20px}
.price-badge{background:linear-gradient(135deg,var(--gold),#a0722a);color:#1a0a2e;font-size:28px;font-weight:900;padding:10px 24px;border-radius:16px;display:inline-block;margin-bottom:20px}
.price-sub{font-size:12px;color:var(--text2);margin-top:4px}

/* Хаан банк заавар */
.khaan-steps{background:var(--surface);border:1px solid var(--border);border-radius:16px;padding:16px;margin-bottom:16px;text-align:left}
.khaan-steps h4{color:var(--lg);font-size:13px;margin-bottom:10px;display:flex;align-items:center;gap:6px}
.step-item{display:flex;align-items:flex-start;gap:10px;margin-bottom:8px;font-size:12px;color:var(--text)}
.step-num{background:var(--gold);color:#1a0a2e;border-radius:50%;width:20px;height:20px;display:flex;align-items:center;justify-content:center;font-weight:900;font-size:11px;flex-shrink:0;margin-top:1px}
.account-copy{background:var(--surface);border:1px solid var(--gold);border-radius:10px;padding:8px 12px;font-size:13px;font-weight:900;color:var(--gold);cursor:pointer;width:100%;margin-bottom:12px;font-family:"Nunito",sans-serif;transition:all .2s}
.account-copy:hover{background:rgba(212,160,23,.1)}

/* Upload хэсэг */
#paymentForm label{display:block;font-size:12px;color:var(--text2);margin-bottom:6px;text-align:left}
#receiptFile{width:100%;padding:10px;border:2px dashed var(--border);border-radius:12px;background:var(--surface);color:var(--text);font-family:"Nunito",sans-serif;font-size:12px;margin-bottom:10px;cursor:pointer}
#receiptFile:hover{border-color:var(--gold)}
.btn-pay{width:100%;background:linear-gradient(135deg,var(--green),var(--lg));color:white;border:none;border-radius:16px;padding:14px;font-family:"Nunito",sans-serif;font-weight:900;font-size:15px;cursor:pointer;transition:all .2s;margin-bottom:8px}
.btn-pay:disabled{opacity:.6}
.btn-close-pay{width:100%;background:var(--surface);border:1px solid var(--border);border-radius:16px;padding:12px;font-family:"Nunito",sans-serif;font-weight:700;font-size:13px;cursor:pointer;color:var(--text)}
#paymentSuccess{display:none;background:rgba(82,183,136,.1);border:1px solid var(--lg);border-radius:16px;padding:16px;margin-bottom:12px}
#paymentSuccess p{color:var(--lg);font-size:13px;line-height:1.7}

/* ── LB & FEEDBACK ── */
.lb-btn-wrap{text-align:center;margin-bottom:12px}
.lb-btn{background:var(--surface);border:1px solid var(--border);border-radius:14px;padding:8px 18px;color:var(--gold);font-family:"Nunito",sans-serif;font-weight:700;font-size:12px;cursor:pointer;transition:all .2s}
.lb-panel{background:var(--surface);border:1px solid var(--border);border-radius:20px;padding:14px;margin-bottom:12px;display:none}
.lb-panel.open{display:block}
.lb-title{font-family:"Dela Gothic One",cursive;font-size:15px;color:var(--gold);margin-bottom:10px;text-align:center}
.lb-row{display:flex;align-items:center;gap:8px;padding:7px 8px;border-radius:12px;margin-bottom:5px;background:rgba(255,255,255,.04)}
.lb-avatar{width:28px;height:28px;border-radius:50%;object-fit:cover;border:2px solid var(--gold);flex-shrink:0}
.lb-rank{font-size:16px;width:24px;text-align:center}
.lb-name{flex:1;font-weight:700;font-size:13px}
.lb-score{font-weight:900;color:var(--gold);font-size:14px}
.lb-date{font-size:10px;color:var(--text2)}
.lb-empty{text-align:center;color:var(--text2);font-size:12px;padding:14px}
.feedback-section{background:var(--surface);border:1px solid var(--border);border-radius:20px;padding:14px;margin-bottom:14px}
.feedback-title{font-weight:900;color:var(--lg);font-size:13px;margin-bottom:8px}
.feedback-form{display:flex;flex-direction:column;gap:7px}
.feedback-form textarea{width:100%;padding:11px;border-radius:14px;border:1px solid var(--border);background:rgba(255,255,255,.06);color:var(--text);font-family:"Nunito",sans-serif;font-size:12px;resize:none;outline:none;height:72px}
.feedback-form textarea:focus{border-color:var(--lg)}
.feedback-form textarea::placeholder{color:var(--text2)}
.btn-feedback{padding:9px 22px;background:linear-gradient(135deg,var(--green),var(--lg));color:white;border:none;border-radius:14px;font-family:"Nunito",sans-serif;font-weight:900;font-size:12px;cursor:pointer;align-self:flex-end}

/* ── TOAST ── */
.feedback-toast{position:fixed;top:20px;left:50%;transform:translateX(-50%) translateY(-120px);padding:12px 26px;border-radius:50px;font-weight:900;font-size:14px;z-index:1000;transition:transform .4s cubic-bezier(.23,1,.32,1);white-space:nowrap}
.feedback-toast.show{transform:translateX(-50%) translateY(0)}
.correct-toast{background:linear-gradient(135deg,#2d6a4f,#52b788);box-shadow:0 8px 25px rgba(82,183,136,.5)}
.wrong-toast{background:linear-gradient(135deg,#7b2d2d,#c1121f);box-shadow:0 8px 25px rgba(193,18,31,.5)}

/* ── COMPLETION ── */
.completion-screen{display:none;text-align:center;padding:32px 20px;position:relative;z-index:10;max-width:700px;margin:0 auto}
.completion-screen.show{display:block}
.big-emoji{font-size:72px;margin-bottom:14px;animation:bounce .6s ease infinite alternate}
.star-rating{font-size:36px;margin-bottom:7px;letter-spacing:4px}
.completion-title{font-family:"Dela Gothic One",cursive;font-size:26px;background:linear-gradient(to right,var(--gold),var(--lg));-webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text;margin-bottom:10px}
.completion-stats{display:flex;justify-content:center;gap:14px;margin:18px 0;flex-wrap:wrap}
.stat-box{background:var(--surface);border:1px solid var(--border);border-radius:18px;padding:12px 18px;min-width:85px}
.stat-box .val{font-size:30px;font-weight:900;color:var(--gold)}
.stat-box .lbl{font-size:10px;color:var(--text2)}
.completion-btns{display:flex;gap:10px;justify-content:center;flex-wrap:wrap}
.btn-restart{padding:13px 32px;background:linear-gradient(135deg,var(--gold),#a0722a);color:#1a0a2e;border:none;border-radius:50px;font-family:"Nunito",sans-serif;font-weight:900;font-size:15px;cursor:pointer;transition:all .2s;box-shadow:0 8px 25px rgba(212,160,23,.4)}
.btn-restart:hover{transform:scale(1.05)}
.btn-lb-view{padding:13px 32px;background:var(--surface);border:1px solid var(--border);color:var(--gold);border-radius:50px;font-family:"Nunito",sans-serif;font-weight:900;font-size:15px;cursor:pointer}
.shuffle-badge{display:inline-flex;align-items:center;gap:4px;background:rgba(212,160,23,.15);border:1px solid rgba(212,160,23,.3);border-radius:10px;padding:3px 10px;font-size:11px;color:var(--gold);font-weight:700;margin-bottom:8px}

/* ── LOCKED CARD ── */
.locked-card{background:var(--surface);border:2px dashed var(--border);border-radius:22px;padding:32px;text-align:center;min-height:200px;display:flex;flex-direction:column;align-items:center;justify-content:center}
.locked-icon{font-size:48px;margin-bottom:12px}
.locked-title{font-family:"Dela Gothic One",cursive;font-size:18px;color:var(--gold);margin-bottom:8px}
.locked-desc{color:var(--text2);font-size:13px;line-height:1.6;margin-bottom:16px}
.btn-unlock{background:linear-gradient(135deg,var(--gold),#a0722a);color:#1a0a2e;border:none;border-radius:16px;padding:12px 28px;font-family:"Nunito",sans-serif;font-weight:900;font-size:14px;cursor:pointer}
</style>
</head>
<body>
<div class="particles" id="particles"></div>
<div class="feedback-toast" id="feedbackToast"></div>

<!-- ═══ НЭВТРЭХ ДЭЛГЭЦ ═══ -->
<div class="auth-screen" id="authScreen">
  <div class="auth-logo">🏛</div>
  <div class="free-badge">🎁 Эхний 20 жор үнэгүй!</div>
  <div class="auth-title">МАНБА ДАЦАН</div>
  <div class="auth-sub">УАУ-ны Эмийн Жорын Тоглоом<br>Монголын уламжлалт эмнэлэг</div>
  <button class="btn-google" id="loginBtn" onclick="signInGoogle()">
    <img src="https://www.gstatic.com/firebasejs/ui/2.0.0/images/auth/google.svg" alt="G">
    Google-ээр нэвтрэх
  </button>
  <div class="auth-note">Нэвтэрснээр та эхний 20 жорыг үнэгүй үзэх боломжтой.<br>21-ээс дээш жорыг сарын 5,000₮-өөр нэвтрэх боломжтой.</div>
</div>

<!-- ═══ ҮНДСЭН ТОГЛООМ ═══ -->
<div id="mainGame" style="display:none">

<!-- HEADER -->
<header>
  <div class="top-bar">
    <div class="user-info">
      <img id="userAvatar" class="user-avatar" src="" alt="">
      <div>
        <div id="userName" class="user-name">Тоглогч</div>
        <div id="premiumBadge" class="premium-badge"></div>
      </div>
    </div>
    <div class="top-actions">
      <button class="icon-btn" onclick="toggleTheme()" id="themeBtn">🌙</button>
      <button class="icon-btn" onclick="toggleShuffle()" id="shuffleBtn">🔀</button>
      <button class="icon-btn" onclick="toggleLB()">🏆</button>
      <button class="icon-btn" onclick="doSignOut()" title="Гарах">🚪</button>
    </div>
  </div>
  <div class="logo-badge">🏛 МАНБА ДАЦАН</div>
  <h1>УАУ-ны Эмийн<br>Жорын Тоглоом</h1>
  <p class="subtitle">Монголын уламжлалт эмнэлэг — 50 жор</p>
</header>

<!-- PREMIUM BANNER -->
<div class="premium-banner" id="premiumBanner">
  <div class="premium-banner-text">
    🔒 21-р жороос эхлэн Premium шаардлагатай<br>
    <span style="font-size:11px;font-weight:400">Сарын ердөө 5,000₮-өөр бүх жорыг нэвтрэ!</span>
  </div>
  <button class="btn-upgrade" onclick="document.getElementById('paywall').classList.add('show')">⭐ Сунгах</button>
</div>

<!-- SCORE -->
<div class="score-bar">
  <div class="score-item gold"><div class="val" id="scoreVal">0</div><div class="lbl">Оноо</div></div>
  <div class="score-item green"><div class="val" id="knownVal">0</div><div class="lbl">Мэднэ</div></div>
  <div class="score-item red"><div class="val" id="unknownVal">0</div><div class="lbl">Мэдэхгүй</div></div>
</div>

<!-- PROGRESS -->
<div class="progress-wrap">
  <div class="progress-track"><div class="progress-fill" id="progressFill" style="width:2%"></div></div>
  <div class="progress-label"><span id="progressText">1 / 50</span><span id="progressPct">2%</span></div>
</div>

<!-- TIMER -->
<div class="timer-bar-wrap" id="timerWrap" style="display:none">
  <div class="timer-bar-track"><div class="timer-bar-fill" id="timerFill"></div></div>
  <div class="timer-label" id="timerLabel">10с</div>
</div>

<!-- LEADERBOARD -->
<div class="card-area" style="padding-bottom:0">
  <div class="lb-btn-wrap">
    <button class="lb-btn" onclick="toggleLB()">🏆 Өдрийн шилдгүүд харах</button>
  </div>
  <div class="lb-panel" id="lbPanel">
    <div class="lb-title">🏆 Өдрийн Шилдгүүд</div>
    <div id="lbList"></div>
  </div>
</div>

<!-- DOTS -->
<div class="dots-row" id="dotsRow"></div>

<div class="card-area">
  <div id="shuffleBadge" class="shuffle-badge" style="display:none">🔀 Холилдсон горим</div>

  <!-- MODE TABS -->
  <div class="mode-tabs">
    <button class="mode-tab active" onclick="setMode('flash')">🃏 Карт</button>
    <button class="mode-tab" onclick="setMode('quiz')">🧠 Тест</button>
    <button class="mode-tab" onclick="setMode('spell')">✏️ Бичих</button>
  </div>

  <!-- FLASHCARD -->
  <div id="flashMode">
    <div class="flashcard-container" onclick="flipCard()">
      <div class="flashcard" id="flashcard">
        <div class="card-face card-front" id="cardFront">
          <div class="card-deco card-deco-1">🌿</div><div class="card-deco card-deco-2">⚗️</div>
          <div class="card-number" id="cardNumber">ЖОР #1</div>
          <div class="card-title" id="cardTitle">Агар 8</div>
          <div class="card-type" id="cardType">/талх/</div>
          <div class="card-ingredients" id="cardIngredients"></div>
          <div class="flip-hint">👆 дарж тайлбар харах</div>
        </div>
        <div class="card-face card-back" id="cardBack">
          <div class="card-deco card-deco-1">💊</div><div class="card-deco card-deco-2">🌱</div>
          <div class="card-number">ТАЙЛБАР</div>
          <div class="card-title-small" id="cardTitleBack"></div>
          <div class="card-description" id="cardDesc"></div>
          <div class="flip-hint" style="margin-top:12px">👆 дарж буцах</div>
        </div>
      </div>
    </div>
    <div class="nav-row">
      <button class="btn-nav" onclick="prevCard()">← Өмнөх</button>
      <span class="card-counter" id="cardCounter">1 / 50</span>
      <button class="btn-nav" onclick="nextCard()">Дараах →</button>
    </div>
    <div class="action-buttons">
      <button class="btn btn-know" id="btnKnow" onclick="markKnown()">✅ Мэднэ</button>
      <button class="btn btn-dontknow" id="btnDontKnow" onclick="markUnknown()">❌ Мэдэхгүй</button>
      <button class="btn btn-next" onclick="nextCard()">→ Дараах карт</button>
    </div>
  </div>

  <!-- QUIZ -->
  <div id="quizMode" style="display:none">
    <div class="quiz-question-box" id="quizQuestion"></div>
    <div class="quiz-options" id="quizOptions"></div>
    <button class="btn-next-full" onclick="nextQuiz()">→ Дараах асуулт</button>
  </div>

  <!-- SPELL -->
  <div id="spellMode" style="display:none">
    <div class="spell-question">
      <strong id="spellClue">Дараах тайлбараас жорын нэрийг тааж бич:</strong>
      <span id="spellHint"></span>
    </div>
    <input type="text" id="spellInput" placeholder="Жорын нэрийг бич..." autocomplete="off" onkeydown="if(event.key==='Enter')checkSpell()">
    <button class="btn-check" onclick="checkSpell()">✓ Шалгах</button>
    <button class="btn-next-full" onclick="nextSpell()">→ Дараах</button>
  </div>

  <!-- FEEDBACK -->
  <div class="feedback-section">
    <div class="feedback-title">📬 Шинэ жор санал болгох</div>
    <div class="feedback-form">
      <textarea id="feedbackText" placeholder="Энд шинэ жор эсвэл санал бичнэ үү..."></textarea>
      <button class="btn-feedback" onclick="sendFeedback()">Илгээх →</button>
    </div>
  </div>
</div>

<!-- COMPLETION -->
<div class="completion-screen" id="completionScreen">
  <div class="big-emoji" id="completionEmoji">🎉</div>
  <div class="star-rating" id="starRating">⭐⭐⭐</div>
  <div class="completion-title" id="completionTitle">Гайхалтай!</div>
  <p style="color:var(--text2);margin-bottom:16px">Та бүх жорыг давж гарлаа!</p>
  <div class="completion-stats">
    <div class="stat-box"><div class="val" id="finalScore">0</div><div class="lbl">Нийт оноо</div></div>
    <div class="stat-box"><div class="val" id="finalKnown">0</div><div class="lbl">Мэдсэн жор</div></div>
    <div class="stat-box"><div class="val" id="finalPct">0%</div><div class="lbl">Амжилт</div></div>
  </div>
  <div class="completion-btns">
    <button class="btn-restart" onclick="restart()">🔄 Дахин тоглох</button>
    <button class="btn-lb-view" onclick="showLBAfterGame()">🏆 Шилдгүүд</button>
  </div>
</div>

</div><!-- /mainGame -->

<!-- ═══ PAYWALL MODAL ═══ -->
<div class="paywall-overlay" id="paywall">
  <div class="paywall-box">
    <div class="paywall-icon">🔒</div>
    <div class="paywall-title">Premium шаардлагатай</div>
    <div class="paywall-desc">21-р жороос эхлэн бүх жорыг үзэхийн тулд сарын эрхийг сунгана уу.</div>
    <div class="price-badge">5,000₮ <span style="font-size:14px;font-weight:400">/сар</span></div>

    <div id="paymentForm">
      <div class="khaan-steps">
        <h4>🏦 Хаан банкаар төлөх заавар</h4>
        <div class="step-item"><div class="step-num">1</div><div>Хаан банкны аппликейшнаа нээнэ үү</div></div>
        <div class="step-item"><div class="step-num">2</div><div>Гуйвуулга → Данс руу шилжүүлэх</div></div>
        <div class="step-item"><div class="step-num">3</div><div>Дараах данс руу <b>5,000₮</b> шилжүүлнэ:</div></div>
        <button class="account-copy" onclick="copyAccount()">
          🏦 Хаан банк: 5022XXXXXX &nbsp;|&nbsp; МАНБА ДАЦАН &nbsp;📋 Хуулах
        </button>
        <div class="step-item"><div class="step-num">4</div><div>Гүйлгээний дэлгэцийн зургийг доор upload хийнэ үү</div></div>
      </div>

      <label>📸 Гүйлгээний баримтын зургийг оруулна уу:</label>
      <input type="file" id="receiptFile" accept="image/*" capture="environment">
      <div id="paymentSuccess">
        <p>✅ <b>Баярлалаа!</b> Таны хүсэлт амжилттай илгээгдлээ.<br>
        Администратор 24 цагийн дотор таны эрхийг идэвхжүүлнэ.<br>
        Имэйлд мэдэгдэл ирнэ.</p>
      </div>
      <button class="btn-pay" id="submitPayBtn" onclick="submitPayment()">✅ Баримт илгээх</button>
    </div>

    <button class="btn-close-pay" onclick="document.getElementById('paywall').classList.remove('show')">✕ Хаах</button>
  </div>
</div>

<script>
const medicines = [
  { id:1, name:"Агар 8", type:"талх", ingredients:"агар, задь, ниншош, жуган, бойгар, руда, арур, нагагзсэр", desc:"Бор саарал өнгөтэй, анхилам үнэртэй, исгэлэн гашуун эхүүн амтай, бүлээн жор. Зүрхэнд мэс туссан, гэмтсэн халуун, хямарсан халуун, галзуурсан, хэлгийрэх, зүрх долгисох зэрэгт тустай.", color:"Бор саарал" },
  { id:2, name:"Агар 15", type:"талх", ingredients:"агар, задь, ниншош, жуган, бойгар, руда, улаан зандан, нагзсэр, 3 үр, мана, гандигар, лидэр, гажаа", desc:"Бор саарал өнгөтэй, содон үнэртэй, ялган амирлуулах бүлээн жор. Хий цус харших, ар өвөрт хатгуулах, ханиалгаж цайвар хөсгөр цэр гарах зэрэгт тустай.", color:"Бор саарал" },
  { id:3, name:"Агар 17", type:"талх", ingredients:"хар агар, задь, ниншош, руда, 3 үр, лидэр, халма шош, дэлүүн шош, ажаг сэржим, ажаг цэрон, лишь, лалапүд, гүгүл, манчин, туулайн зурх", desc:"Бор саарал өнгөтэй, гашуун амттай, өвөрмөц үнэртэй бүлээн жор. Хий цус харшсанаас зүрх дэлсэх, толгой эргэх, муу цус бөөрөнд буух зэрэгт тустай.", color:"Бор саарал" },
  { id:4, name:"Агар 35", type:"талх", ingredients:"хар агар, цагаан агар, улаан агар, руда, сороол, жуган, гүргүм, задь, лишь, сүгмэл, гагол, юмдүүжин, дэгд, норов-7, ниншош, хонлин, япон зоосон цэцэг, ажагцэрон, манчин, заар, их мах, гүгүл, бойгар, нагагзсэр, улаан зандан, цагаан зандан, хүнцэл, сэмбрүү, сэржмядаг", desc:"Шаргал өнгөтэй, эхүүн үнэртэй, гашуун амттай, тэгш чадалтай, өчүүхэн бүлээн жор. Нян, халуун, хий зэрэгт тустай.", color:"Шаргал" },
  { id:5, name:"Арур 10", type:"талх", ingredients:"арур, сүгмэл, брагшин, гүргүм, шүгцэр, шунхан, жажиг, зод, халмашош, дэгд", desc:"Шар ногоон өнгөтэй, өвөрмөц үнэртэй, гашуун амттай, өчүүхэн сэрүүн жор бөгөөд туулгах үйлдэл бүхий жор. Бөөрний гэмтсэн халуун, шээс чавдагших, ууц нуруугаар өвдөх зэрэгт тустай.", color:"Шар ногоон" },
  { id:6, name:"Арчун", type:"талх", ingredients:"арур, сүгмэл, брагшин, гүргүм, шүгцэр, шунхан, жажиг, зод, халмашош, дэгд, шудаг, манчин, руда, ларьз", desc:"Ногоон шаргал өнгөтэй, өвөрмөц үнэртэй, гашуун амттай өчүүхэн сэрүүн бөгөөд туулгах үйлдэл бүхий жор. Бөөрний гэмтсэн халуун, шээс чавдагших зэрэгт тустай.", color:"Ногоон шаргал" },
  { id:7, name:"Арьсны түхлэг", type:"талх", ingredients:"бойгор, талгадорж, сомаранз, юнва, шар мод, руда, дүгмонин, тогоны хөө, нишинханд, бор бурам", desc:"Ногоон өнгөтэй, өвөрмөц үнэртэй, амтлаг эхүүн амтай, бага зэрэг бүлээн жор. Арьсны өвчнийг эцэслэн арилгана.", color:"Ногоон" },
  { id:8, name:"Бамнад 3", type:"тан", ingredients:"гахайн цус, сэмбрүү, бриангу", desc:"Ногоон өнгөтэй, исгэлэн гашуун амттай, бүлээн жор. Хорны өвчин ба хөлийн бам өвчин, чийгний өвчнийг арилгана.", color:"Ногоон" },
  { id:9, name:"Банзи 12", type:"талх", ingredients:"банздо, дагш, бонгар, зэмбэ, барбада, цагаан зандан, жуган, гүргүм, гүгүл, заар, барагшин, гиван", desc:"Бор шаргал өнгөтэй, өвөрмөц үнэртэй, гашуун амттай, сэрүүн бөгөөд бага зэрэг хоруу чанартай. Нян хижиг, ханиад, хүч ихтэй шарын халуун, сахуу, боом, дуу хаагдах зэрэгт тустай.", color:"Бор шаргал" },
  { id:10, name:"Барагшин 5", type:"тан", ingredients:"барагшин, балига, гүргүм, юмдүүжин, домти", desc:"Ногоон шаргал өнгөтэй, анхилам үнэртэй, бага зэрэг амтлаг эхүүн амттай, сэрүүн жор. Элгээр өвдөж эвэршсэх, элэг томрох, цэзгээр хатгуулах зэрэгт тустай.", color:"Ногоон шаргал" },
  { id:11, name:"Барагшин 9", type:"талх", ingredients:"барагшин, гүргүм, домти, сүгмэл, хүдрийн заар, бриангу, бонгар, арур, мэхзэр", desc:"Бор ногоон өнгөтэй, өвөрмөц үнэртэй, бага зэрэг гашуун амттай, сэрүүн жор. Ходоодны цус шарын өвчин, ходоод тэлгэдэх, ходоод гэдсэнд хижигийн халуун буугаад бөөлжих, суулгахыг арилгана.", color:"Бор ногоон" },
  { id:12, name:"Барагшин ханд", type:"ханд эм", ingredients:"барагшин", desc:"Цэвэршүүлсэн барагшин нь өтгөн гилгэр хар өнгөтэй, өвөрмөц үнэртэй, амтлаг амттай, халуун буурах нөлөөтэй. Ходоод элгийн бүхний халуун чанартай өвчний үед хоолны өмнө хойно хоёуланд буцалгаж уух.", color:"Хар" },
  { id:13, name:"Бойгор 10", type:"талх", ingredients:"бойгар, талгадорж, сомаранз, лидэр, руда, юмдүүжин, 3 үр, барагшин", desc:"Бор саарал өнгөтэй, өвөрмөц үнэртэй, гашуун амттай, ялган хатаах тэгш жор.", color:"Бор саарал" },
  { id:14, name:"Бойчун 10", type:"талх", ingredients:"бойгар, талгадорж, сомаранз, лидэр, руда, юмдүүжин, 3 үр, барагшин", desc:"Бор саарал өнгөтэй, өвөрмөц үнэртэй, гашуун амттай, ялган хатаах тэгш жор. Хэрэх, тулай, халуун шар усны өвчин, шархирах, уе мэч өвдөхийг анагаана.", color:"Бор саарал" },
  { id:15, name:"Бойчун 15", type:"талх", ingredients:"бойгар, талгадорж, сомаранз, лидэр, руда, юмдүүжин, 3 үр, ба, шудаг, манчин, ларьз, сэндэнханд", desc:"Шар ус татрахгүйд Бойчун-15 эмийг сэндэнчитангаар даруулж ууж болно.", color:"Шаргал" },
  { id:16, name:"Болман 7", type:"талх", ingredients:"гүргүм, сэржмядаг, бонгар, юмдүүжин, жүрүр, бриангу, хөх удвал", desc:"Бор саарал өнгөтэй, хүрэн, тулай, бөөрний хэрэх, уе мэчний өвчнийг анагаана. Мөн халуун усан хаванг хүйтэнд урвулах чадалтай.", color:"Бор саарал" },
  { id:17, name:"Брайву 3", type:"тан", ingredients:"арур, барур, жүрүр", desc:"Шаргал өнгөтэй, сул үнэртэй, эхүүн бага зэрэг гашуун амттай, ялгах сэрүүн жор. Хижиг болон хямарсан халуунд тустай. Цус ялган амирлуулах зорилгоор Брайву-3 тангаар даруулж ууна.", color:"Шаргал" },
  { id:18, name:"Брэга 13", type:"талх", ingredients:"брига, сүгмэл, доод 3 үр, шүгцэр, шунхан, жажиг, зод, халмашош, арур", desc:"Бор хүрэн өнгөтэй, өвөрмөц үнэртэй, эхүүн амттай, ялган амирлуулах өчүүхэн сэрүүн жор. Давсагны өвчин язгуурт, бөөр гэмтсэн доргисон, халуун хүйтэн хий хуралдсан, хавдсан хавдрыг арилгана.", color:"Бор хүрэн" },
  { id:19, name:"Вүдод", type:"талх", ingredients:"сэма, цэнэ, библин", desc:"Шаргал өнгөтэй, халуун амтлаг амттай, өвөрмөц анхилуун үнэртэй, бүлээн жор. Хүүхэд олохыг хүссэнд тустай. Эмчийн заалтаар хэрэглэх бөгөөд орой унтахын өмнө буцалсан бүлээнээр усаар даруулж ууна.", color:"Шаргал" },
  { id:20, name:"Вүтов", type:"талх", ingredients:"гаа, библин, повари, руда 2, арур, руда", desc:"Шар өнгөтэй, халуун амттай, сул үнэртэй, бүлээн жор. Хүүхэд олохыг хүссэнд тустай. Эмчийн заалтаар хэрэглэх бөгөөд өглөө өлөн элэн буцалсан бүлээнээр усаар даруулж ууна.", color:"Шар" },
  { id:21, name:"Вонтог 25", type:"талх", ingredients:"илжигний цус, цагаан зандан, улаан зандан, арур, барур, жүрүр, жуган, лишь, задь, хөх сүгмэл, гагол, бойгар, талгадорж, сомаранз, нагагзсэр, банжингарво, банздо, давсан, хөх дэгд, башга, лидэр, гүргүм, ларьз", desc:"Хүрэн өнгөтэй, сулхан үнэртэй, гашуун амттай. Тунгалаг цэвийн боловсролтыг тэгшитгэх, халуун шар усыг хатаах, тулай өвчнийг анагаана.", color:"Хүрэн" },
  { id:22, name:"Гавар 9", type:"талх", ingredients:"гавар, жуган, агар, цагаан зандан, улаан зандан, бонгар, банзидо, барбада", desc:"Хүрэн саарал өнгөтэй, өвөрмөц үнэртэй, эхүүн амттай. Тархсан, хямарсан, дэлгэрсэн, эс боловсорсон халуун, цэзгээр хатгуулах, улаан шар буюу утааны өнгөтэй цэр гарахыг анагаана.", color:"Хүрэн саарал" },
  { id:23, name:"Гагол 11", type:"талх", ingredients:"гагол, арур, жажиг, сэржмядаг, мана, зээргэнэ, жумз, руда, зод, явагчара, сүгмэл", desc:"Бор шаргал өнгөтэй, өвөрмөц анхилам үнэртэй, давслаг бага зэрэг амтлаг амттай, тэгш жор. Дэлүүний боловсорсон өвчнийг анагаана.", color:"Бор шаргал" },
  { id:24, name:"Гарнаг 10", type:"талх", ingredients:"сэмбрүү, шинца, сүгмэл, библин, арур, жамц, сэржмядаг, дүгмонин, домти, пагрил", desc:"Хар өнгөтэй, сул үнэртэй бага зэрэг давслаг амттай, бүлээн жор. Хийн өвчин, эм эс шингэсэн, бадган бэтэг тэргүүт бусдын эрхэнд хавсарсан шар бүгдийг анагаана.", color:"Хар" },
  { id:25, name:"Гарша 6", type:"талх", ingredients:"сүгмэл, гаа, библин, чихэр өвс, арур, гожил", desc:"Шаргал өнгөтэй, халуун амттай, бүлээн жор. Шанбрам өвчнийг арилгана. Өглөө, орой идэзний өмнө 2 тунг буцалсан усаар даруулж уух.", color:"Шаргал" },
  { id:26, name:"Гиван 9", type:"талх", ingredients:"гиван, юмдүүжин, барагшин, удвал, руда, сэржмядаг, дэгд, балига, гүргүм", desc:"Улаан шаргал өнгөтэй, содон үнэргүй, гашуун амттай, сэрүүн жор. Элэг бэртэж гэмтсэн, элгэнд цус дэлгэрсэн, элгний бадган дэлгэрснийг анагаана.", color:"Улаан шаргал" },
  { id:27, name:"Гоньд 6", type:"талх", ingredients:"гоньд, халгайн үр, цээнэ, мухар цагаан, дангун, сармис", desc:"Цайвар шаргал өнгөтэй, өвөрмөц үнэртэй бүлээн жор. Толгой эргэх, бие махбод хүндрэх, хөөстэй бөөлжих, идээнд дурлагүй болох, бадган хавсарсан өвчинд тустай.", color:"Цайвар шаргал" },
  { id:28, name:"Гоюу 7", type:"талх", ingredients:"гоюу, библин, цагаан гаа, жац, сүгмэл, сэмбрүү, шинца", desc:"Хүрэн улаан өнгөтэй, анхилам үнэртэй, халуун амттай, бүлээн жор. Бөөр давсагны өвчин бүгдийг анагаана. Ялангуяа доодод оршсон хүйтэн хий хавсарсан өвдэлт ихтэй өвчин, эмэгтэйн цагаан юм гарах өвчинд зэрэгт тустай.", color:"Хүрэн улаан" },
  { id:29, name:"Гүнбрүм 7", type:"талх", ingredients:"үзэм, жуган, гүргүм, чихэр өвс, мэхзэр, сэмбрүү, шинца", desc:"Улаан шаргал өнгөтэй, содон үнэргүй, амтлаг амттай, бүлээн жор. Уушгины хүйтэн өвчин, амьсгаадах, цэр хөвхорходоо бэрх болохыг анагаана.", color:"Улаан шаргал" },
  { id:30, name:"Гүнбрүм 11", type:"талх", ingredients:"үзэм, жуган, гүргүм, чихэр өвс, мэхзэр, сэмбрүү, шинца, лишь, банжингарав, руда", desc:"Цайвар шаргал өнгөтэй, содон үнэргүй, амтлаг амттай, бүлээн жор. Уушгины өвчин амьсгаадах, цэр хөвхорходоо бэрх болохыг анагаана.", color:"Цайвар шаргал" },
  { id:31, name:"Гүргүм 7", type:"талх", ingredients:"гүргүм, жуган, удвал, балига, дэгд, арур, зээргэнэ", desc:"Бор шаргал өнгөтэй, анхилам үнэртэй, амтлаг эхүүн бага зэрэг гашуун амттай, сэрүүн жор. Элгний шинэ хуучин өвчин, элэг бэртэж гэмтэх, элгний цус дэлгэрэх, нүд болон биеийг шарлуулах зэрэгт тустай.", color:"Бор шаргал" },
  { id:32, name:"Гүрчун", type:"талх", ingredients:"гүргүм, жуган, удвал, балига, дэгд, арур, зээргэнэ, шудаг, манчин, руда, ларьз", desc:"Бор шаргал өнгөтэй, анхилам үнэртэй, амтлаг эхүүн бага зэрэг гашуун амттай, сэрүүн жор. Элгний шинэ хуучин өвчин, шинэ хуучин хүйтэн өвчнийг анагаана.", color:"Бор шаргал" },
  { id:33, name:"Гүрчун 11", type:"талх", ingredients:"гүргүм, жуган, удвал, балига, дэгд, арур, зээргэнэ, шудаг, манчин, руда, ларьз", desc:"Элгний шинэ хуучин өвчин, элэг бэртэж гэмтэх, элгний цус дэлгэрэх, нүд болон биеийг шарлуулах зэрэгт тустай. Нян, ад хавсарсан, амирлахад бэрхтэй үед Чун-5 нэмбэл Гүрчун болно.", color:"Бор шаргал" },
  { id:34, name:"Гүргүм 13", type:"талх", ingredients:"гүргүм, жуган, цала, улаан зандан, заар, бонгор, жамбрай, руда, арур, барур, жүрүр", desc:"Улаан хүрэн өнгөтэй, өчүүхэн анхилам үнэртэй, гашуун бага зэрэг исгэлэн амттай, сэрүүн жор. Элэг доройтох, найруулсан хор, бөөр гэмтэж бэртэх, шижин халуунаар хөөх зэрэгт тустай.", color:"Улаан хүрэн" },
  { id:35, name:"Гүргүмчигтан", type:"тан", ingredients:"гүргүм", desc:"Улаан хүрэн өнгөтэй анхилам үнэртэй, амтлаг амттай, сэрүүн жор. Элгний өвчнийг анагаах ба буюу цусыг тогтооход үйлдэлтэй. Өдөр, шөнө дунд идэш шингэсэн үед, өвчин боссон үед 1-2 тунг буцалгаж уух.", color:"Улаан хүрэн" },
  { id:36, name:"Дагш ханд", type:"ханд эм", ingredients:"дагш", desc:"Ногоон өнгөтэй, зуурмал байдалтай байна. Шарханд гаднаас элдэв халдвар орохоос сэргийлэх, тургэн эдгэрэж үйлчилгээтэй түрхлэг. Шарханд гаднаас тургэн хэрэглэнэ.", color:"Ногоон" },
  { id:37, name:"Даль 16", type:"талх", ingredients:"сэмбрүү, шинца, сүгмэл, библин, гүргүм, руда, лишь, мэхзэр, ниншош, үзэм, чихэр өвс, жуган, даль дэгсрэн, арнаг, задь", desc:"Шар цайвар өнгөтэй, содон үнэргүй, гашуун бага зэрэг амтлаг амттай, бүлээн эм. Бие махбодийг тэгшлэн хэсэгт тустай.", color:"Шар цайвар" },
  { id:38, name:"Данманайжог", type:"талх", ingredients:"сэмбрүү, шинца, сүгмэл, библин, гүргүм", desc:"Улаан шаргал өнгөтэй, анхилам үнэртэй, халуун амттай, бүлээн жор. Ходоодны галын илчийг үүсгэх, эс шингэсэн өвчин, доод биеийн хүйтэн, халуун хүйтэн харшилдсан өвчнийг анагаана.", color:"Улаан шаргал" },
  { id:39, name:"Данмачудэн", type:"талх", ingredients:"сэмбрүү, шинца, сүгмэл, библин, үсү, жүрүр, жамба, дэгсрэн", desc:"Бор шаргал өнгөтэй, содон үнэргүй, давслаг халуун амттай, бүлээн жор. Усны 18 өвчнийг татаж хатаана.", color:"Бор шаргал" },
  { id:40, name:"Дарву 5", type:"талх", ingredients:"чацаргана, үзэм, чихэр өвс, жүрүр, руда", desc:"Шар өнгөтэй, зугтгааш байдалтай, гашуун бага зэрэг исгэлэн амтыг мэдрэгдэхгүй тэгш жор. Уушгины өвчинд тустай.", color:"Шар" },
  { id:41, name:"Дигаг 3", type:"тан", ingredients:"жумз, арур, хужир", desc:"Шаргал өнгөтэй, анхилам үнэртэй, давслаг эхүүн амттай, бүлээн жор. Шимшин 3 гэсэн өөр нэг нэртэй. Өтгөн хатсны чийглэнэ.", color:"Шаргал" },
  { id:42, name:"Диман 4", type:"тан", ingredients:"банжин гарво, царван, жамц, зэмбэ", desc:"Ногоон өнгөтэй, хурц үнэртэй, давслаг гашуун амттай, сэрүүн тэгш жор. Хоолой хатах, хоолойн халуунын дарна.", color:"Ногоон" },
  { id:43, name:"Диман 12", type:"талх", ingredients:"банздо, банжин гарво, жилж омбо, дагш, хөх тоосго, зэмбэ, гүгүл, арур, шудаг, барагшин, башга, жуган, бримог", desc:"Барааан бор өнгөтэй, содон үнэргүй, гашуун амттай, сэрүүн жор. Хоолойн өвчин, хоолойн булчирхайн өвчнийг анагаана.", color:"Барааан бор" },
  { id:44, name:"Димантан", type:"тан", ingredients:"бонгарын навч, чихэр өвс, жуган, хөх тоосго", desc:"Ногоон өнгөтэй, анхилам үнэртэй, амтлаг амттай, сэрүүн жор. Хоолойн өвчин, сахуу, боом, дуу хаагдсан, нян өвчин хоолойд буусны анагаана.", color:"Ногоон" },
  { id:45, name:"Доншин 4", type:"тан", ingredients:"доншин, мэхзэр, шингүн, арур", desc:"Цайвар шаргал өнгөтэй, бага зэрэг эмхий, шингүний үнэртэй, эхүүн амттай, бүлээн жор. Хий хавсарсан хижиг ба хүйтэн хий арилгаж, тугнээс гүүсэн нүүр ам муруйхыг арилгана.", color:"Цайвар шаргал" },
  { id:46, name:"Донжүгохов", type:"талх", ingredients:"жамц, шудаг, шунхан, юнгар", desc:"Бараан шаргал өнгөтэй, өвөрмөц хурц үнэртэй, давслаг эхүүн амттай, түрхлэгийн бүлээн жор. Нүүрэнд гарсан санх, оройд буцалсан бүлээн усанд дэвтээн зуурч нимгэн маск хэлбэрээр 30 минут орчим тавина.", color:"Бараан шаргал" },
  { id:47, name:"Доржжан", type:"талх", ingredients:"манчин, жац, шүгцэр, оим, бамбай, гүргүм, гиван, заар, домти", desc:"Бор шаргал өнгөтэй, содон үнэргүй, давслаг амттай, тэгш жор. Шээс хаагдсан, дусагнасан, самс дэлгэрсэн зэрэгт машид сайн.", color:"Бор шаргал" },
  { id:48, name:"Дүгсэлтан", type:"тан", ingredients:"хирсний эвэр, жамьянмядаг, чихэр өвс, туйплан, хонлин", desc:"Шаргал өнгөтэй, шаар баргийн хортой, өчүүхэн сэрүүн жор. Хорыг арилгаж, хордлогыг тайлна.", color:"Шаргал" },
  { id:49, name:"Дүдзи 5", type:"дэвтээлэг", ingredients:"агь, шүгцэр, сургар, балган бургас, зээргэнэ", desc:"Ногоон өнгөтэй, өвөрмөц үнэртэй, бүлээн жор. Уе гишүү атийж хөшсөн, хатгаж доголсон, хавдар боом, шархны эс боловсорсон, шунаж хуучирсан, хавдсан үед шар ус гүйж зэрэгт тустай.", color:"Ногоон" },
  { id:50, name:"Дүдзи 10", type:"талх", ingredients:"зэмбэ, дэгд, юмдүүжин, банздо, мана, лидэр, гажа, гандигар, арур, чихэр өвс", desc:"Цайвар ногоон өнгөтэй, анхилам үнэртэй, амтлаг амттай, боловсруулан арилгах бага зэргийн сэрүүн жор. Нянгийн халуун, хижигийн халуун, ханиад бүгдэд рашаан адил.", color:"Цайвар ногоон" },
];

// ═══ STATE ═══
let currentIndex=0, score=0, known=0, unknown=0;
let mode="flash", isFlipped=false;
let knownCards=new Set(), unknownCards=new Set();
let quizAnswered=false, spellAnswered=false;
let shuffleMode=false, shuffledOrder=[];
let timerInterval=null, timerSeconds=10;
let isDark=true;
const FREE_LIMIT=20;

function initGame() {
  initParticles(); initOrder(); initDots(); updateCard(); updateScore();
  document.getElementById("themeBtn").textContent = "🌙";
  document.getElementById("shuffleBtn").style.opacity = "0.5";
}

// ═══ THEME ═══
function toggleTheme() {
  isDark=!isDark;
  document.documentElement.setAttribute("data-theme", isDark?"dark":"light");
  document.getElementById("themeBtn").textContent = isDark?"🌙":"☀️";
}

// ═══ SHUFFLE ═══
function initOrder() {
  shuffledOrder = medicines.map((_,i)=>i);
  if(shuffleMode) shuffledOrder.sort(()=>Math.random()-.5);
}
function toggleShuffle() {
  shuffleMode=!shuffleMode;
  document.getElementById("shuffleBtn").style.opacity = shuffleMode?"1":"0.5";
  document.getElementById("shuffleBadge").style.display = shuffleMode?"inline-flex":"none";
  initOrder(); currentIndex=0; updateCard(); initDots();
}
function currentMed() { return medicines[shuffledOrder[currentIndex]]; }

// ═══ PAYWALL CHECK ═══
function isLocked(idx) { return !isPremium && idx >= FREE_LIMIT; }

// ═══ PARTICLES ═══
function initParticles() {
  const c=document.getElementById("particles"); c.innerHTML="";
  const colors=["#D4A017","#52b788","#48cae4","#ff9f1c"];
  for(let i=0;i<18;i++){
    const p=document.createElement("div"); p.className="particle";
    const s=Math.random()*5+3;
    p.style.cssText=`width:${s}px;height:${s}px;left:${Math.random()*100}%;background:${colors[i%4]};animation-duration:${Math.random()*15+10}s;animation-delay:${Math.random()*10}s`;
    c.appendChild(p);
  }
}

// ═══ DOTS ═══
function initDots() {
  const row=document.getElementById("dotsRow"); row.innerHTML="";
  medicines.forEach((_,i)=>{
    const d=document.createElement("div");
    const real=shuffledOrder[i];
    d.className="mini-dot " + (i===currentIndex?"current": isLocked(i)?"locked": knownCards.has(real)?"known": unknownCards.has(real)?"unknown":"unseen");
    row.appendChild(d);
  });
}
function updateDots() {
  document.querySelectorAll(".mini-dot").forEach((d,i)=>{
    d.className="mini-dot";
    const real=shuffledOrder[i];
    if(i===currentIndex) d.classList.add("current");
    else if(isLocked(i)) d.classList.add("locked");
    else if(knownCards.has(real)) d.classList.add("known");
    else if(unknownCards.has(real)) d.classList.add("unknown");
    else d.classList.add("unseen");
  });
}

// ═══ CARD ═══
function updateCard() {
  if(isLocked(currentIndex)) { showLockedCard(); return; }
  hideLockedCard();
  const m=currentMed();
  document.getElementById("cardNumber").textContent="ЖОР #"+m.id;
  document.getElementById("cardTitle").textContent=m.name;
  document.getElementById("cardType").textContent="/"+m.type+"/";
  document.getElementById("cardIngredients").textContent=m.ingredients;
  document.getElementById("cardDesc").textContent=m.desc;
  document.getElementById("cardTitleBack").textContent=m.name;
  document.getElementById("cardCounter").textContent=(currentIndex+1)+" / "+medicines.length;
  isFlipped=false;
  document.getElementById("flashcard").classList.remove("flipped");
  updateProgress(); updateDots();
}

function showLockedCard() {
  document.getElementById("flashMode").innerHTML=`
    <div class="locked-card">
      <div class="locked-icon">🔒</div>
      <div class="locked-title">Premium шаардлагатай</div>
      <div class="locked-desc">Та эхний ${FREE_LIMIT} жорыг үзлээ!<br>Бүх жорыг үзэхийн тулд сарын эрхийг сунгана уу.</div>
      <button class="btn-unlock" onclick="document.getElementById('paywall').classList.add('show')">⭐ 5,000₮-өөр сунгах</button>
    </div>`;
  document.getElementById("cardCounter").textContent=(currentIndex+1)+" / "+medicines.length;
  updateProgress(); updateDots();
}
function hideLockedCard() {
  if(!document.getElementById("flashcard")) {
    document.getElementById("flashMode").innerHTML=`
    <div class="flashcard-container" onclick="flipCard()">
      <div class="flashcard" id="flashcard">
        <div class="card-face card-front" id="cardFront">
          <div class="card-deco card-deco-1">🌿</div><div class="card-deco card-deco-2">⚗️</div>
          <div class="card-number" id="cardNumber"></div>
          <div class="card-title" id="cardTitle"></div>
          <div class="card-type" id="cardType"></div>
          <div class="card-ingredients" id="cardIngredients"></div>
          <div class="flip-hint">👆 дарж тайлбар харах</div>
        </div>
        <div class="card-face card-back" id="cardBack">
          <div class="card-deco card-deco-1">💊</div><div class="card-deco card-deco-2">🌱</div>
          <div class="card-number">ТАЙЛБАР</div>
          <div class="card-title-small" id="cardTitleBack"></div>
          <div class="card-description" id="cardDesc"></div>
          <div class="flip-hint" style="margin-top:12px">👆 дарж буцах</div>
        </div>
      </div>
    </div>
    <div class="nav-row">
      <button class="btn-nav" onclick="prevCard()">← Өмнөх</button>
      <span class="card-counter" id="cardCounter"></span>
      <button class="btn-nav" onclick="nextCard()">Дараах →</button>
    </div>
    <div class="action-buttons">
      <button class="btn btn-know" id="btnKnow" onclick="markKnown()">✅ Мэднэ</button>
      <button class="btn btn-dontknow" id="btnDontKnow" onclick="markUnknown()">❌ Мэдэхгүй</button>
      <button class="btn btn-next" onclick="nextCard()">→ Дараах карт</button>
    </div>`;
  }
}

function updateProgress() {
  const pct=Math.round(((currentIndex+1)/medicines.length)*100);
  document.getElementById("progressFill").style.width=pct+"%";
  document.getElementById("progressText").textContent=(currentIndex+1)+" / "+medicines.length;
  document.getElementById("progressPct").textContent=pct+"%";
}
function updateScore() {
  document.getElementById("scoreVal").textContent=score;
  document.getElementById("knownVal").textContent=known;
  document.getElementById("unknownVal").textContent=unknown;
}
function flipCard() {
  if(isLocked(currentIndex)) return;
  isFlipped=!isFlipped;
  const fc=document.getElementById("flashcard");
  const front=fc.querySelector(".card-front"), back=fc.querySelector(".card-back");
  const hiding=isFlipped?front:back, showing=isFlipped?back:front;
  hiding.style.opacity="0";
  setTimeout(()=>{ fc.classList.toggle("flipped",isFlipped); hiding.style.opacity=""; showing.style.opacity="0"; setTimeout(()=>{showing.style.opacity=""},30); },180);
}

// ═══ FLASH ACTIONS ═══
function markKnown() {
  const ri=shuffledOrder[currentIndex];
  if(!knownCards.has(ri)){knownCards.add(ri);unknownCards.delete(ri);known++;score+=10;updateScore();showToast("✅ Сайн байна! +10 оноо","correct");document.getElementById("btnKnow").classList.add("correct-anim");setTimeout(()=>document.getElementById("btnKnow").classList.remove("correct-anim"),700);}
  nextCard();
}
function markUnknown() {
  const ri=shuffledOrder[currentIndex];
  if(!unknownCards.has(ri)){unknownCards.add(ri);knownCards.delete(ri);unknown++;updateScore();showToast("📚 Дахин давтаарай!","wrong");document.getElementById("btnDontKnow").classList.add("wrong-anim");setTimeout(()=>document.getElementById("btnDontKnow").classList.remove("wrong-anim"),600);}
  nextCard();
}
function prevCard() {
  if(currentIndex>0){currentIndex--;updateCard();if(mode==="quiz")setupQuiz();if(mode==="spell")setupSpell();}
}
function nextCard() {
  stopTimer();
  if(currentIndex<medicines.length-1){currentIndex++;updateCard();if(mode==="quiz")setupQuiz();if(mode==="spell")setupSpell();}
  else showCompletion();
}

// ═══ MODE ═══
function setMode(m) {
  mode=m;
  document.querySelectorAll(".mode-tab").forEach((t,i)=>t.classList.toggle("active",["flash","quiz","spell"][i]===m));
  document.getElementById("flashMode").style.display=m==="flash"?"block":"none";
  document.getElementById("quizMode").style.display=m==="quiz"?"block":"none";
  document.getElementById("spellMode").style.display=m==="spell"?"block":"none";
  document.getElementById("timerWrap").style.display=(m==="quiz"||m==="spell")?"block":"none";
  stopTimer();
  if(m==="quiz")setupQuiz(); if(m==="spell")setupSpell();
}

// ═══ TIMER ═══
function startTimer() {
  stopTimer(); timerSeconds=10;
  const fill=document.getElementById("timerFill"), lbl=document.getElementById("timerLabel");
  fill.style.width="100%"; fill.style.background="var(--lg)"; lbl.style.color="var(--lg)"; lbl.textContent="10с";
  timerInterval=setInterval(()=>{
    timerSeconds-=0.1;
    fill.style.width=(timerSeconds/10*100)+"%"; lbl.textContent=Math.ceil(timerSeconds)+"с";
    if(timerSeconds<=3){fill.style.background="#c1121f";lbl.style.color="#c1121f";}
    else if(timerSeconds<=6){fill.style.background="#f77f00";lbl.style.color="#f77f00";}
    if(timerSeconds<=0){stopTimer();timeUp();}
  },100);
}
function stopTimer(){if(timerInterval){clearInterval(timerInterval);timerInterval=null;}}
function timeUp() {
  if(mode==="quiz"&&!quizAnswered){quizAnswered=true;showToast("⏰ Хугацаа дууссан!","wrong");document.querySelectorAll(".quiz-option").forEach(b=>{if(b.dataset.correct==="1")b.classList.add("correct");});unknown++;unknownCards.add(currentIndex);updateScore();updateDots();}
  else if(mode==="spell"&&!spellAnswered){spellAnswered=true;const m=currentMed();document.getElementById("spellInput").style.borderColor="#c1121f";showToast("Хариулт: "+m.name,"wrong");unknown++;unknownCards.add(currentIndex);updateScore();updateDots();}
}

// ═══ QUIZ ═══
function setupQuiz() {
  if(isLocked(currentIndex)){document.getElementById("quizQuestion").textContent="🔒 Premium шаардлагатай";document.getElementById("quizOptions").innerHTML="";return;}
  quizAnswered=false;
  const m=currentMed();
  const qTypes=[
    {q:`"${m.name}" жор ямар өнгөтэй вэ?`,correct:m.color,getWrong:()=>medicines.filter(x=>x.color!==m.color).map(x=>x.color)},
    {q:`"${m.name}" жор ямар төрлийн эм вэ?`,correct:m.type,getWrong:()=>medicines.filter(x=>x.type!==m.type).map(x=>x.type)},
    {q:`Дараах орц нь ямар жорынх вэ?\n"${m.ingredients}"`,correct:m.name,getWrong:()=>medicines.filter(x=>x.name!==m.name).map(x=>x.name)},
  ];
  const qt=qTypes[currentIndex%qTypes.length];
  document.getElementById("quizQuestion").textContent=qt.q;
  let wrongs=[...new Set(qt.getWrong())].sort(()=>Math.random()-.5).slice(0,3);
  let opts=[qt.correct,...wrongs].sort(()=>Math.random()-.5);
  const c=document.getElementById("quizOptions"); c.innerHTML="";
  opts.forEach(opt=>{
    const btn=document.createElement("button"); btn.className="quiz-option"; btn.textContent=opt; btn.dataset.correct=opt===qt.correct?"1":"0";
    btn.onclick=()=>selectQuiz(btn,opt,qt.correct); c.appendChild(btn);
  });
  startTimer();
}
function selectQuiz(btn,selected,correct) {
  if(quizAnswered)return; quizAnswered=true; stopTimer();
  const ri=shuffledOrder[currentIndex];
  if(selected===correct){btn.classList.add("correct");score+=15;known++;knownCards.add(ri);updateScore();showToast("🎯 Зөв! +15 оноо","correct");}
  else{btn.classList.add("wrong");unknown++;unknownCards.add(ri);updateScore();showToast("❌ Буруу!","wrong");document.querySelectorAll(".quiz-option").forEach(b=>{if(b.dataset.correct==="1")b.classList.add("correct");});}
  updateDots();
}
function nextQuiz(){nextCard();}

// ═══ SPELL ═══
function setupSpell() {
  if(isLocked(currentIndex)){document.getElementById("spellHint").textContent="🔒 Premium шаардлагатай";document.getElementById("spellInput").disabled=true;return;}
  document.getElementById("spellInput").disabled=false;
  spellAnswered=false; const m=currentMed();
  document.getElementById("spellHint").textContent=m.desc;
  document.getElementById("spellInput").value=""; document.getElementById("spellInput").style.borderColor="var(--border)";
  startTimer();
}
function checkSpell() {
  if(spellAnswered)return; spellAnswered=true; stopTimer();
  const m=currentMed(), input=document.getElementById("spellInput").value.trim().toLowerCase(), correct=m.name.toLowerCase(), inp=document.getElementById("spellInput");
  const ri=shuffledOrder[currentIndex];
  if(input===correct||(correct.includes(input)&&input.length>3)){inp.style.borderColor="var(--lg)";score+=20;known++;knownCards.add(ri);updateScore();showToast("🌟 Гайхалтай! +20 оноо","correct");inp.classList.add("correct-anim");setTimeout(()=>inp.classList.remove("correct-anim"),700);}
  else{inp.style.borderColor="#c1121f";showToast("Хариулт: "+m.name,"wrong");unknown++;unknownCards.add(ri);updateScore();inp.classList.add("wrong-anim");setTimeout(()=>inp.classList.remove("wrong-anim"),600);}
  updateDots();
}
function nextSpell(){nextCard();}

// ═══ TOAST ═══
function showToast(msg,type) {
  const t=document.getElementById("feedbackToast"); t.textContent=msg;
  t.className=`feedback-toast ${type==="correct"?"correct-toast":"wrong-toast"} show`;
  setTimeout(()=>t.classList.remove("show"),2200);
}

// ═══ LEADERBOARD ═══
function toggleLB() {
  const p=document.getElementById("lbPanel"); p.classList.toggle("open");
  if(p.classList.contains("open")) {
    if(window.__firebase) window.__firebase.loadFirestoreLB();
    else renderLocalLB();
  }
}
function renderLocalLB() {
  try{
    const lb=JSON.parse(localStorage.getItem("manba_lb")||"[]");
    const today=new Date().toLocaleDateString("mn-MN");
    const medals=["🥇","🥈","🥉"];
    const html=lb.filter(r=>r.date===today).map((r,i)=>`<div class="lb-row"><div class="lb-rank">${medals[i]||(i+1)}</div><div><div class="lb-name">${r.name}</div><div class="lb-date">${r.date}</div></div><div class="lb-score">${r.score} оноо</div></div>`).join("")||"<div class='lb-empty'>Өнөөдрийн бичлэг байхгүй</div>";
    document.getElementById("lbList").innerHTML=html;
  }catch(e){}
}
function showLBAfterGame(){document.getElementById("completionScreen").classList.remove("show");document.querySelector(".card-area").style.display="block";document.querySelector(".score-bar").style.display="flex";document.querySelector(".progress-wrap").style.display="block";const p=document.getElementById("lbPanel");p.classList.add("open");toggleLB();p.scrollIntoView({behavior:"smooth"});}

// ═══ FEEDBACK ═══
function sendFeedback() {
  const txt=document.getElementById("feedbackText").value.trim();
  if(!txt)return;
  const saved=JSON.parse(localStorage.getItem("manba_feedback")||"[]");
  saved.push({name:window.playerName||"Нэргүй",text:txt,date:new Date().toLocaleDateString("mn-MN")});
  localStorage.setItem("manba_feedback",JSON.stringify(saved));
  document.getElementById("feedbackText").value="";
  showToast("📬 Санал амжилттай хадгалагдлаа!","correct");
}

// ═══ COPY ACCOUNT ═══
function copyAccount() {
  navigator.clipboard.writeText("5022XXXXXX").then(()=>showToast("📋 Данс хуулагдлаа!","correct"));
}

// ═══ COMPLETION ═══
function showCompletion() {
  stopTimer();
  document.querySelector(".card-area").style.display="none";
  document.querySelector(".score-bar").style.display="none";
  document.querySelector(".progress-wrap").style.display="none";
  document.getElementById("timerWrap").style.display="none";
  const pct=Math.round((known/medicines.length)*100);
  document.getElementById("finalScore").textContent=score;
  document.getElementById("finalKnown").textContent=known;
  document.getElementById("finalPct").textContent=pct+"%";
  let emoji,stars,title;
  if(pct>=90){emoji="🏆";stars="⭐⭐⭐⭐⭐";title="Гайхалтай! Та эмч болно!";}
  else if(pct>=70){emoji="🎉";stars="⭐⭐⭐⭐";title="Маш сайн! Үргэлжлүүлэ!";}
  else if(pct>=50){emoji="💪";stars="⭐⭐⭐";title="Сайн! Дахин давтаарай!";}
  else{emoji="📚";stars="⭐⭐";title="Дахин суралц!";}
  document.getElementById("completionEmoji").textContent=emoji;
  document.getElementById("starRating").textContent=stars;
  document.getElementById("completionTitle").textContent=title;
  document.getElementById("completionScreen").classList.add("show");
  if(window.__firebase) window.__firebase.saveScoreFirestore(score);
  const today=new Date().toLocaleDateString("mn-MN");
  const lb=JSON.parse(localStorage.getItem("manba_lb")||"[]");
  lb.push({name:window.playerName||"Нэргүй",score,date:today});
  lb.sort((a,b)=>b.score-a.score);
  localStorage.setItem("manba_lb",JSON.stringify(lb.slice(0,20)));
}

function restart() {
  currentIndex=0;score=0;known=0;unknown=0;
  knownCards=new Set();unknownCards=new Set();
  quizAnswered=false;spellAnswered=false;
  stopTimer(); initOrder(); updateScore(); updateCard(); initDots();
  document.getElementById("completionScreen").classList.remove("show");
  document.querySelector(".card-area").style.display="block";
  document.querySelector(".score-bar").style.display="flex";
  document.querySelector(".progress-wrap").style.display="block";
  setMode("flash");
}

let isPremium = false;
</script>
</body>
</html>
