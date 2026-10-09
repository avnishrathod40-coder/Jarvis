# Jarvis
pluginManagement { repositories { google(); mavenCentral(); gradlePluginPortal() } }
dependencyResolutionManagement { repositories { google(); mavenCentral() } }
rootProject.name = "Jarvis"
include(":app")
plugins {
 id("com.android.application") version "8.5.2" apply false
 id("org.jetbrains.kotlin.android") version "1.9.24" apply false
}
org.gradle.jvmargs=-Xmx2g
android.nonTransitiveRClass=true
plugins {
 id("com.android.application")
 id("org.jetbrains.kotlin.android")
}
android {
 namespace = "com.aman.jarvis"
 compileSdk = 34
 defaultConfig {
 applicationId = "com.aman.jarvis"
 minSdk = 26
 targetSdk = 34
 versionCode = 1
 versionName = "1.0"
 }
 compileOptions {
 sourceCompatibility = JavaVersion.VERSION_17
 targetCompatibility = JavaVersion.VERSION_17
 }
 kotlinOptions { jvmTarget = "17" }
}
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
 <uses-permission android:name="android.permission.RECORD_AUDIO" />
 <uses-permission android:name="android.permission.INTERNET" />
 <uses-permission android:name="android.permission.CALL_PHONE" />
 <uses-permission android:name="android.permission.READ_CONTACTS" />
 <uses-permission android:name="com.android.alarm.permission.SET_ALARM" />
 <queries>
 <intent><action android:name="android.intent.action.MAIN" /><category android:name="android.intent.category.LAUNCHER" /></intent>
 <intent><action android:name="android.speech.RecognitionService" /></intent>
 </queries>
 <application
 android:allowBackup="false"
 android:icon="@android:drawable/ic_btn_speak_now"
 android:label="Jarvis"
 android:theme="@android:style/Theme.Material.NoActionBar">
 <activity android:name=".MainActivity" android:exported="true" android:windowSoftInputMode="adjustResize">
 <intent-filter>
 <action android:name="android.intent.action.MAIN" />
 <category android:name="android.intent.category.LAUNCHER" />
 </intent-filter>
 </activity>
 </application>
</manifest>
package com.aman.jarvis
import android.Manifest
import android.animation.ValueAnimator
import android.app.Activity
import android.app.AlertDialog
import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import android.content.pm.PackageManager
import android.content.pm.ResolveInfo
import android.graphics.Color
import android.graphics.Typeface
import android.graphics.drawable.GradientDrawable
import android.hardware.camera2.CameraCharacteristics
import android.hardware.camera2.CameraManager
import android.media.AudioManager
import android.net.Uri
import android.os.BatteryManager
import android.os.Bundle
import android.os.Handler
import android.os.Looper
import android.provider.AlarmClock
import android.provider.ContactsContract
import android.provider.Settings
import android.speech.RecognitionListener
import android.speech.RecognizerIntent
import android.speech.SpeechRecognizer
import android.speech.tts.TextToSpeech
import android.speech.tts.UtteranceProgressListener
import android.view.Gravity
import android.view.KeyEvent
import android.view.View
import android.view.inputmethod.EditorInfo
import android.widget.Button
import android.widget.EditText
import android.widget.LinearLayout
import android.widget.ScrollView
import android.widget.TextView
import org.json.JSONArray
import org.json.JSONObject
import java.net.HttpURLConnection
import java.net.URL
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale
import kotlin.concurrent.thread
class MainActivity : Activity(), TextToSpeech.OnInitListener {
 private val CYAN = Color.parseColor("#00F0FF")
 private val MAG = Color.parseColor("#FF2BD6")
 private val BG = Color.parseColor("#04050D")
 private val ui = Handler(Looper.getMainLooper())
 private val history = JSONArray()
 private val prefs by lazy { getSharedPreferences("jarvis", Context.MODE_PRIVATE) }
 private lateinit var tts: TextToSpeech
 private lateinit var status: TextView
 private lateinit var logView: TextView
 private lateinit var scroll: ScrollView
 private lateinit var orb: View
 private lateinit var input: EditText
 private var sr: SpeechRecognizer? = null
 private var pulse: ValueAnimator? = null
 private var listening = false
 private var speaking = false
 private var wake = true
 private val ALIAS = linkedMapOf(
 "\u092F\u0942\u091F\u094D\u092F\u0942\u092C" to "youtube", "\u0935\u094D\u0939\u093E\u091F\u094D\u0938\u090F\u092A" to "whatsapp", "\u0935\u0949\u091F\u094D\u0938\u0910\u092A" to "whatsapp", "\u0907\u0902\u0938\u094D\u091F\u093E\u0917\u094D\u0930\u093E\u092E" to "instagram",
 "\u091C\u0940\u092E\u0947\u0932" to "gmail", "\u092E\u0948\u092A\u094D\u0938" to "maps", "\u0915\u0948\u092E\u0930\u093E" to "camera", "\u0938\u0947\u091F\u093F\u0902\u0917\u094D\u0938" to "settings", "\u0915\u094D\u0930\u094B\u092E" to "chrome",
 "\u0915\u0948\u0932\u0915\u0941\u0932\u0947\u091F\u0930" to "calculator", "\u0917\u0948\u0932\u0930\u0940" to "gallery", "\u0938\u094D\u092A\u0949\u091F\u093F\u092B\u093E\u0908" to "spotify", "\u0916\u094B\u0932\u094B" to "kholo", "\u0916\u094B\u0932" to "kholo",
 "\u0905\u0932\u093E\u0930\u094D\u092E" to "alarm", "\u091F\u093E\u0907\u092E\u0930" to "timer", "\u091F\u0949\u0930\u094D\u091A" to "torch", "\u0915\u0949\u0932" to "call", "\u092C\u0948\u091F\u0930\u0940" to "battery"
 )
 private fun dp(v: Int) = (v * resources.displayMetrics.density).toInt()
 private fun orbDrawable(c: Int): GradientDrawable {
 val g = GradientDrawable()
 g.setShape(GradientDrawable.OVAL)
 g.setGradientType(GradientDrawable.RADIAL_GRADIENT)
 g.setGradientRadius(dp(95).toFloat())
 g.setColors(intArrayOf(Color.WHITE, c, Color.parseColor("#0A2A9A"), Color.TRANSPARENT))
 return g
 }
 private fun neon(label: String): Button {
 val b = Button(this)
 b.text = label
 b.isAllCaps = false
 b.setTextColor(CYAN)
 b.textSize = 13f
 val g = GradientDrawable()
 g.setColor(Color.TRANSPARENT)
 g.setStroke(dp(1), CYAN)
 g.setCornerRadius(dp(22).toFloat())
 b.background = g
 return b
 }
 override fun onCreate(b: Bundle?) {
 super.onCreate(b)
 window.statusBarColor = BG
 window.navigationBarColor = BG
 val root = LinearLayout(this)
 root.orientation = LinearLayout.VERTICAL
 root.gravity = Gravity.CENTER_HORIZONTAL
 root.setBackgroundColor(BG)
 root.setPadding(dp(16), dp(40), dp(16), dp(16))
 val title = TextView(this)
 title.text = "J.A.R.V.I.S"
 title.textSize = 26f
 title.setTextColor(CYAN)
 title.typeface = Typeface.MONOSPACE
 title.letterSpacing = 0.3f
 title.setShadowLayer(24f, 0f, 0f, CYAN)
 root.addView(title)
 orb = View(this)
 orb.background = orbDrawable(CYAN)
 val olp = LinearLayout.LayoutParams(dp(190), dp(190))
 olp.topMargin = dp(24)
 olp.bottomMargin = dp(12)
 root.addView(orb, olp)
 orb.setOnClickListener { toggleListening() }
 status = TextView(this)
 status.setTextColor(CYAN)
 status.textSize = 14f
 status.gravity = Gravity.CENTER
 root.addView(status)
 val row = LinearLayout(this)
 row.orientation = LinearLayout.HORIZONTAL
 val wakeBtn = neon("Wake word: ON")
 wakeBtn.setOnClickListener {
 wake = !wake
 wakeBtn.text = "Wake word: " + if (wake) "ON" else "OFF"
 }
 val keyBtn = neon("AI Key")
 keyBtn.setOnClickListener { keyDialog() }
 val blp = LinearLayout.LayoutParams(0, LinearLayout.LayoutParams.WRAP_CONTENT, 1f)
 blp.setMargins(dp(4), dp(8), dp(4), dp(8))
 row.addView(wakeBtn, blp)
 row.addView(keyBtn, blp)
 root.addView(row, LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, LinearLayout.LayoutParams.WRAP_CONTENT))
 scroll = ScrollView(this)
 logView = TextView(this)
 logView.setTextColor(Color.parseColor("#D6F7FF"))
 logView.textSize = 14f
 logView.typeface = Typeface.MONOSPACE
 scroll.addView(logView)
 root.addView(scroll, LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, 0, 1f))
 input = EditText(this)
 input.hint = "Ya yahan type karo..."
 input.setHintTextColor(Color.parseColor("#5C7A8A"))
 input.setTextColor(Color.WHITE)
 input.isSingleLine = true
 input.imeOptions = EditorInfo.IME_ACTION_SEND
 input.setOnEditorActionListener { _, _, _ -> submit(); true }
 root.addView(input, LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, LinearLayout.LayoutParams.WRAP_CONTENT))
 setContentView(root)
 pulse = ValueAnimator.ofFloat(0.88f, 1.12f).apply {
 duration = 2400
 repeatCount = ValueAnimator.INFINITE
 repeatMode = ValueAnimator.REVERSE
 addUpdateListener { orb.scaleX = it.animatedValue as Float; orb.scaleY = it.animatedValue as Float }
 start()
 }
 setMode(0)
 tts = TextToSpeech(this, this)
 }
 override fun onInit(s: Int) {
 tts.setPitch(0.85f)
 tts.setOnUtteranceProgressListener(object : UtteranceProgressListener() {
 override fun onStart(id: String?) {}
 override fun onDone(id: String?) { ui.post { speaking = false; setMode(if (listening) 1 else 0); startRec() } }
 @Deprecated("Deprecated in Java")
 override fun onError(id: String?) { onDone(id) }
 })
 say("Systems online. Orb dabao aur bolo, Jarvis, youtube kholo.")
 }
 override fun onResume() { super.onResume(); if (listening && !speaking) startRec() }
 override fun onPause() { super.onPause(); sr?.cancel() }
 override fun onDestroy() { super.onDestroy(); sr?.destroy(); tts.shutdown() }
 override fun onRequestPermissionsResult(code: Int, p: Array<out String>, r: IntArray) {
 super.onRequestPermissionsResult(code, p, r)
 if (code == 1) {
 if (r.isNotEmpty() && r[0] == PackageManager.PERMISSION_GRANTED) { setMode(1); startRec() }
 else { listening = false; say("Mic permission chahiye. Settings mein allow karo.") }
 }
 }
 private fun setMode(m: Int) {
 orb.background = orbDrawable(if (m == 2) MAG else CYAN)
 pulse?.duration = when (m) { 1 -> 800L; 2 -> 450L; else -> 2400L }
 status.text = when (m) { 1 -> "Listening..."; 2 -> "Speaking..."; else -> "Offline. Orb dabao." }
 }
 private fun log(who: String, msg: String) {
 logView.append(who + ": " + msg + "\n\n")
 scroll.post { scroll.fullScroll(View.FOCUS_DOWN) }
 }
 private fun submit() {
 val s = input.text.toString().trim()
 if (s.isNotEmpty()) { input.setText(""); run(s) }
 }
 private fun toggleListening() {
 if (listening) { listening = false; sr?.cancel(); setMode(0); return }
 listening = true
 if (checkSelfPermission(Manifest.permission.RECORD_AUDIO) == PackageManager.PERMISSION_GRANTED) { setMode(1); startRec() }
 else requestPermissions(arrayOf(Manifest.permission.RECORD_AUDIO), 1)
 }
 private fun say(t: String) {
 log("Jarvis", t)
 speaking = true
 sr?.cancel()
 setMode(2)
 tts.setLanguage(if (t.any { it in '\u0900'..'\u097F' }) Locale("hi", "IN") else Locale("en", "IN"))
 tts.speak(t, TextToSpeech.QUEUE_FLUSH, null, "j")
 }
 private fun startRec() {
 if (!listening || speaking) return
 if (!SpeechRecognizer.isRecognitionAvailable(this)) { listening = false; say("Is phone par speech recognition available nahi hai."); return }
 sr?.destroy()
 sr = SpeechRecognizer.createSpeechRecognizer(this)
 sr?.setRecognitionListener(object : RecognitionListener {
 override fun onReadyForSpeech(p: Bundle?) {}
 override fun onBeginningOfSpeech() {}
 override fun onRmsChanged(v: Float) {}
 override fun onBufferReceived(b: ByteArray?) {}
 override fun onEndOfSpeech() {}
 override fun onEvent(t: Int, p: Bundle?) {}
 override fun onPartialResults(p: Bundle?) {
 p?.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)?.firstOrNull()?.let { if (!speaking) status.text = it }
 }
 override fun onError(e: Int) {
 if (e == SpeechRecognizer.ERROR_INSUFFICIENT_PERMISSIONS) { listening = false; setMode(0); return }
 ui.postDelayed({ startRec() }, 600)
 }
 override fun onResults(r: Bundle?) {
 val t = r?.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)?.firstOrNull()
 if (t != null) handle(t)
 ui.postDelayed({ startRec() }, 400)
 }
 })
 val i = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH)
 i.putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
 i.putExtra(RecognizerIntent.EXTRA_LANGUAGE, "hi-IN")
 i.putExtra(RecognizerIntent.EXTRA_PARTIAL_RESULTS, true)
 sr?.startListening(i)
 }
 private fun handle(text: String) {
 var t = text.trim()
 if (wake) {
 val w = Regex("(jarvis|jarvish|jarvas|\u091C\u093E\u0930\u094D\u0935\u093F\u0938|\u091C\u093E\u0930\u0935\u093F\u0938|\u091C\u0930\u094D\u0935\u093F\u0938|\u091C\u093E\u0930\u094D\u0935\u093F\u0936|\u091C\u093E\u0930\u0935\u0938)", RegexOption.IGNORE_CASE)
 if (!w.containsMatchIn(t)) return
 t = t.replace(w, "").trim()
 if (t.isEmpty()) { say("Ji sir, boliye."); return }
 }
 run(t)
 }
 private fun go(i: Intent, msg: String) {
 say(msg)
 ui.postDelayed({
 try { i.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK); startActivity(i) }
 catch (e: Exception) { say("Ye kaam is phone par nahi ho paaya.") }
 }, 1200)
 }
 private fun view(url: String) = Intent(Intent.ACTION_VIEW, Uri.parse(url))
 private fun findApp(name: String): ResolveInfo? {
 val list = packageManager.queryIntentActivities(Intent(Intent.ACTION_MAIN).addCategory(Intent.CATEGORY_LAUNCHER), 0)
 val n = name.lowercase(Locale.ROOT).trim()
 fun label(r: ResolveInfo) = r.loadLabel(packageManager).toString().lowercase(Locale.ROOT)
 return list.firstOrNull { label(it) == n } ?: list.firstOrNull { label(it).contains(n) }
 }
 private fun contactNumber(name: String): String? {
 contentResolver.query(
 ContactsContract.CommonDataKinds.Phone.CONTENT_URI,
 arrayOf(ContactsContract.CommonDataKinds.Phone.NUMBER),
 ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME + " LIKE ?", arrayOf("%$name%"), null
 )?.use { c -> if (c.moveToFirst()) return c.getString(0) }
 return null
 }
 private fun dial(num: String) {
 if (checkSelfPermission(Manifest.permission.CALL_PHONE) == PackageManager.PERMISSION_GRANTED)
 go(Intent(Intent.ACTION_CALL, Uri.parse("tel:$num")), "Call laga raha hoon.")
 else {
 requestPermissions(arrayOf(Manifest.permission.CALL_PHONE), 3)
 go(Intent(Intent.ACTION_DIAL, Uri.parse("tel:$num")), "Dialer khol raha hoon.")
 }
 }
 private fun torch(on: Boolean) {
 try {
 val cm = getSystemService(Context.CAMERA_SERVICE) as CameraManager
 val id = cm.cameraIdList.firstOrNull { cm.getCameraCharacteristics(it).get(CameraCharacteristics.FLASH_INFO_AVAILABLE) == true }
 if (id == null) { say("Is phone mein flash nahi mila."); return }
 cm.setTorchMode(id, on)
 say(if (on) "Torch on." else "Torch off.")
 } catch (e: Exception) { say("Torch nahi chal payi.") }
 }
 private fun mediaKey(code: Int) {
 val am = getSystemService(Context.AUDIO_SERVICE) as AudioManager
 am.dispatchMediaKeyEvent(KeyEvent(KeyEvent.ACTION_DOWN, code))
 am.dispatchMediaKeyEvent(KeyEvent(KeyEvent.ACTION_UP, code))
 }
 private fun run(raw: String) {
 log("You", raw)
 var t = raw.lowercase(Locale.ROOT).replace(Regex("[.,!?]"), " ").replace(Regex("\\s+"), " ").trim()
 for ((k, v) in ALIAS) t = t.replace(k, v)
 fun has(vararg w: String) = w.any { t.contains(it) }
 if (t == "stop" || t == "chup" || t == "\u091A\u0941\u092A") { tts.stop(); speaking = false; return }
 val num = Regex("\\+?\\d[\\d\\s-]{6,}\\d").find(t)
 if (num != null) {
 val digits = num.value.replace(Regex("[\\s-]"), "")
 val after = t.substring(num.range.last + 1).trim()
 if (has("call", "dial", "phone")) return dial(digits)
 if (has("whatsapp")) {
 val d = digits.replace("+", "").let { if (it.length == 10) "91$it" else it }
 return go(view("https://wa.me/$d?text=" + Uri.encode(after)), "WhatsApp khol raha hoon.")
 }
 if (has("sms", "text", "message"))
 return go(Intent(Intent.ACTION_SENDTO, Uri.parse("smsto:$digits")).putExtra("sms_body", after), "Message taiyaar hai.")
 }
 val cn = Regex("^(?:call|phone|dial)\\s+(?:karo\\s+)?(.+)").find(t) ?: Regex("^(.+?)\\s+ko\\s+(?:call|phone)").find(t)
 if (cn != null) {
 if (checkSelfPermission(Manifest.permission.READ_CONTACTS) != PackageManager.PERMISSION_GRANTED) {
 requestPermissions(arrayOf(Manifest.permission.READ_CONTACTS), 2)
 return say("Contacts permission do, phir dobara bolo.")
 }
 val n = contactNumber(cn.groupValues[1])
 return if (n != null) dial(n) else say(cn.groupValues[1] + " naam ka contact nahi mila.")
 }
 val tm = Regex("(\\d+)\\s*(second|sec|\u0938\u0947\u0915\u0902\u0921|minute|min|\u092E\u093F\u0928\u091F|hour|ghant|\u0918\u0902\u091F)").find(t)
 if (tm != null && has("timer", "remind")) {
 val n = tm.groupValues[1].toInt()
 val u = tm.groupValues[2]
 val s = if (u.startsWith("hour") || u.startsWith("ghant") || u.startsWith("\u0918\u0902\u091F")) n * 3600 else if (u.startsWith("min") || u.startsWith("\u092E\u093F\u0928\u091F")) n * 60 else n
 return go(Intent(AlarmClock.ACTION_SET_TIMER).putExtra(AlarmClock.EXTRA_LENGTH, s).putExtra(AlarmClock.EXTRA_SKIP_UI, true), "Timer set kar diya.")
 }
 if (has("alarm", "jagao", "uthao")) {
 val m = Regex("(\\d{1,2})(?:[: ](\\d{2}))?\\s*(am|pm|baje|\u092C\u091C\u0947)").find(t)
 ?: return say("Alarm ka time bolo, jaise 6 30 am ka alarm.")
 var h = m.groupValues[1].toInt()
 val min = m.groupValues[2].ifEmpty { "0" }.toInt()
 if ((m.groupValues[3] == "pm" || has("shaam", "raat")) && h < 12) h += 12
 return go(Intent(AlarmClock.ACTION_SET_ALARM).putExtra(AlarmClock.EXTRA_HOUR, h).putExtra(AlarmClock.EXTRA_MINUTES, min).putExtra(AlarmClock.EXTRA_SKIP_UI, true), "Alarm set kar diya.")
 }
 if (has("torch", "flashlight", "flash light")) return torch(!has("off", "band", "\u092C\u0902\u0926"))
 if (has("volume", "awaaz", "\u0906\u0935\u093E\u091C")) {
 val am = getSystemService(Context.AUDIO_SERVICE) as AudioManager
 val max = am.getStreamMaxVolume(AudioManager.STREAM_MUSIC)
 val p = Regex("(\\d+)").find(t)
 when {
 p != null -> am.setStreamVolume(AudioManager.STREAM_MUSIC, p.value.toInt().coerceIn(0, 100) * max / 100, AudioManager.FLAG_SHOW_UI)
 has("up", "badha", "zyada", "increase", "\u0924\u0947\u091C") -> am.adjustStreamVolume(AudioManager.STREAM_MUSIC, AudioManager.ADJUST_RAISE, AudioManager.FLAG_SHOW_UI)
 has("down", "kam", "ghata", "lower", "decrease", "\u0927\u0940\u092E\u0940") -> am.adjustStreamVolume(AudioManager.STREAM_MUSIC, AudioManager.ADJUST_LOWER, AudioManager.FLAG_SHOW_UI)
 has("max", "full") -> am.setStreamVolume(AudioManager.STREAM_MUSIC, max, AudioManager.FLAG_SHOW_UI)
 has("mute", "band", "zero") -> am.setStreamVolume(AudioManager.STREAM_MUSIC, 0, AudioManager.FLAG_SHOW_UI)
 else -> return say("Volume kitna karna hai? Jaise volume 50 percent.")
 }
 return say("Volume set.")
 }
 when (t) {
 "pause", "ruko", "pause music", "stop music" -> { mediaKey(KeyEvent.KEYCODE_MEDIA_PAUSE); return }
 "resume", "play music", "music chalao", "gaana chalao" -> { mediaKey(KeyEvent.KEYCODE_MEDIA_PLAY); return }
 "next", "next song", "agla", "agla gaana" -> { mediaKey(KeyEvent.KEYCODE_MEDIA_NEXT); return }
 "previous", "previous song", "pichla gaana" -> { mediaKey(KeyEvent.KEYCODE_MEDIA_PREVIOUS); return }
 }
 if (has("battery")) {
 val b = registerReceiver(null, IntentFilter(Intent.ACTION_BATTERY_CHANGED))
 val lvl = (b?.getIntExtra(BatteryManager.EXTRA_LEVEL, 0) ?: 0) * 100 / (b?.getIntExtra(BatteryManager.EXTRA_SCALE, 100) ?: 100)
 val st = b?.getIntExtra(BatteryManager.EXTRA_STATUS, -1)
 val ch = st == BatteryManager.BATTERY_STATUS_CHARGING || st == BatteryManager.BATTERY_STATUS_FULL
 return say("Battery $lvl percent hai" + if (ch) ", charging chal rahi hai." else ".")
 }
 if (has("time", "samay", "baje kya", "\u091F\u093E\u0907\u092E", "\u0938\u092E\u092F")) return say("Abhi " + SimpleDateFormat("h:mm a", Locale.ENGLISH).format(Date()) + " ho rahe hain.")
 if (has("date", "tarikh", "aaj", "\u0924\u093E\u0930\u0940\u0916", "\u0906\u091C")) return say("Aaj " + SimpleDateFormat("EEEE, d MMMM yyyy", Locale.ENGLISH).format(Date()) + " hai.")
 val panel = when {
 has("wifi", "\u0935\u093E\u0908\u092B\u093E\u0908", "\u0935\u093E\u0908 \u092B\u093E\u0908") -> Settings.ACTION_WIFI_SETTINGS
 has("bluetooth", "\u092C\u094D\u0932\u0942\u091F\u0942\u0925") -> Settings.ACTION_BLUETOOTH_SETTINGS
 has("brightness", "display") -> Settings.ACTION_DISPLAY_SETTINGS
 has("location", "gps") -> Settings.ACTION_LOCATION_SOURCE_SETTINGS
 has("airplane", "flight mode") -> Settings.ACTION_AIRPLANE_MODE_SETTINGS
 else -> null
 }
 if (panel != null && has("on", "off", "chalu", "band", "kholo", "settings", "karo"))
 return go(Intent(panel), "Settings khol raha hoon, wahan ek tap se on ya off kar lo.")
 val nv = Regex("(?:navigate to|navigate|directions to|route to|rasta|le chalo)\\s*(?:to\\s+)?(.+)").find(t)
 if (nv != null) return go(view("google.navigation:q=" + Uri.encode(nv.groupValues[1])), "Route nikaal raha hoon.")
 val pl = Regex("^(?:play|chalao|bajao)\\s+(.+)").find(t) ?: Regex("^(.+?)\\s+(?:chalao|bajao|play karo)$").find(t)
 if (pl != null) return go(view("https://www.youtube.com/results?search_query=" + Uri.encode(pl.groupValues[1])), "YouTube par " + pl.groupValues[1] + " chala raha hoon.")
 val om = Regex("^(?:open|kholo|launch|start)\\s+(.+)").find(t) ?: Regex("^(.+?)\\s+(?:kholo|open karo|open|chalu karo)$").find(t)
 if (om != null) {
 val name = om.groupValues[1].replace(Regex("^(?:the|app)\\s+"), "")
 val ri = findApp(name)
 val li = ri?.let { packageManager.getLaunchIntentForPackage(it.activityInfo.packageName) }
 return if (li != null) go(li, "$name khol raha hoon.") else say("$name naam ka app nahi mila.")
 }
 val sr2 = Regex("^(?:search|google|dhundo|dhoondo)\\s+(?:karo\\s+|for\\s+)?(.+)").find(t)
 if (sr2 != null) return go(view("https://www.google.com/search?q=" + Uri.encode(sr2.groupValues[1])), "Search kar raha hoon.")
 if (t.matches(Regex("(hello|hi|hey|namaste|kaise ho|kya haal hai)"))) return say("Namaste sir. Systems online. Kya karna hai?")
 if (has("who are you", "tum kaun", "kaun ho")) return say("Main Jarvis hoon, aapka personal voice assistant.")
 if (has("joke", "chutkula")) return say(listOf(
 "Computer ne glasses kyun pehne? Kyunki usse better Windows chahiye thi.",
 "Programmer ghar late kyun aaya? Bug mil gaya tha, aur chai nahi."
 ).random())
 val key = prefs.getString("key", "") ?: ""
 if (key.isNotEmpty()) askClaude(t, key)
 else go(view("https://www.google.com/search?q=" + Uri.encode(t)), "Samajh nahi aaya, Google par dhundh raha hoon. AI dimaag ke liye AI Key button se key daalo.")
 }
 private fun askClaude(q: String, key: String) {
 status.text = "Thinking..."
 history.put(JSONObject().put("role", "user").put("content", q))
 while (history.length() > 10) { history.remove(0); history.remove(0) }
 thread {
 var ok = false
 val reply = try {
 val c = URL("https://api.anthropic.com/v1/messages").openConnection() as HttpURLConnection
 c.requestMethod = "POST"
 c.connectTimeout = 15000
 c.readTimeout = 40000
 c.setRequestProperty("x-api-key", key)
 c.setRequestProperty("anthropic-version", "2023-06-01")
 c.setRequestProperty("content-type", "application/json")
 c.doOutput = true
 val body = JSONObject()
 .put("model", "claude-sonnet-5-5")
 .put("max_tokens", 300)
 .put("system", "You are Jarvis, a witty voice assistant for Aman. Reply in Hinglish (Hindi and English mix) in at most 3 short sentences. Plain speech only: no markdown, no lists, no emojis.")
 .put("messages", history)
 c.outputStream.use { it.write(body.toString().toByteArray()) }
 val code = c.responseCode
 val txt = (if (code in 200..299) c.inputStream else c.errorStream).bufferedReader().readText()
 if (code in 200..299) { ok = true; JSONObject(txt).getJSONArray("content").getJSONObject(0).getString("text") }
 else "AI error $code. API key check karo."
 } catch (e: Exception) { "Internet ya AI connection mein dikkat hai." }
 ui.post {
 if (ok) history.put(JSONObject().put("role", "assistant").put("content", reply))
 say(reply)
 }
 }
 }
 private fun keyDialog() {
 val e = EditText(this)
 e.hint = "sk-ant-..."
 e.setText(prefs.getString("key", ""))
 AlertDialog.Builder(this)
 .setTitle("Anthropic API key")
 .setView(e)
 .setPositiveButton("Save") { _, _ ->
 prefs.edit().putString("key", e.text.toString().trim()).apply()
 say("AI dimaag chalu ho gaya.")
 }
 .setNegativeButton("Cancel", null)
 .show()
 }
}
name: Build APK
on:
 workflow_dispatch:
 push:
 branches: [main]
jobs:
 build:
 runs-on: ubuntu-latest
 steps:
 - uses: actions/checkout@v4
 - uses: actions/setup-java@v4
 with:
 distribution: temurin
 java-version: 17
 - uses: gradle/actions/setup-gradle@v4
 with:
 gradle-version: 8.7
 - run: gradle assembleDebug --no-daemon
 - uses: actions/upload-artifact@v4
 with:
 name: Jarvis-APK
 path: app/build/outputs/apk/debug/app-debug.apk
