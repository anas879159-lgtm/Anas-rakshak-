#anas raksha-
name: Build ANAS RAKSHA APK

on:
  push:
  workflow_dispatch:

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: App ka code banao
        run: |
          set -e
          mkdir -p raksha
          cat > "raksha/settings.gradle" <<'RAKSHA_EOF'
          pluginManagement {
              repositories { google(); mavenCentral(); gradlePluginPortal() }
          }
          dependencyResolutionManagement {
              repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
              repositories { google(); mavenCentral() }
          }
          rootProject.name = "AnasRaksha"
          include ':app'
          RAKSHA_EOF
          cat > "raksha/build.gradle" <<'RAKSHA_EOF'
          plugins {
              id 'com.android.application' version '8.5.2' apply false
          }
          RAKSHA_EOF
          cat > "raksha/gradle.properties" <<'RAKSHA_EOF'
          org.gradle.jvmargs=-Xmx2048m
          android.useAndroidX=true
          RAKSHA_EOF
          mkdir -p "raksha/app"
          cat > "raksha/app/build.gradle" <<'RAKSHA_EOF'
          plugins {
              id 'com.android.application'
          }

          android {
              namespace 'com.anas.raksha'
              compileSdk 34

              defaultConfig {
                  applicationId "com.anas.raksha"
                  minSdk 24
                  targetSdk 34
                  versionCode 1
                  versionName "1.0"
              }

              buildTypes {
                  release { minifyEnabled false }
              }

              compileOptions {
                  sourceCompatibility JavaVersion.VERSION_17
                  targetCompatibility JavaVersion.VERSION_17
              }
          }
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main"
          cat > "raksha/app/src/main/AndroidManifest.xml" <<'RAKSHA_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">

              <uses-permission android:name="android.permission.SEND_SMS" />
              <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
              <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
              <uses-permission android:name="android.permission.CALL_PHONE" />
              <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
              <uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
              <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
              <uses-permission android:name="android.permission.VIBRATE" />
              <uses-permission android:name="android.permission.WAKE_LOCK" />

              <uses-feature android:name="android.hardware.telephony" android:required="false" />
              <uses-feature android:name="android.hardware.camera.flash" android:required="false" />

              <application
                  android:allowBackup="true"
                  android:icon="@drawable/ic_launcher"
                  android:roundIcon="@drawable/ic_launcher"
                  android:label="@string/app_name"
                  android:theme="@style/AppTheme">

                  <activity
                      android:name=".MainActivity"
                      android:exported="true"
                      android:screenOrientation="portrait">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>

                  <activity
                      android:name=".FakeCallActivity"
                      android:exported="false"
                      android:screenOrientation="portrait"
                      android:showWhenLocked="true"
                      android:turnScreenOn="true" />

                  <service
                      android:name=".RakshaService"
                      android:exported="false"
                      android:foregroundServiceType="location" />
              </application>
          </manifest>
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/java/com/anas/raksha"
          cat > "raksha/app/src/main/java/com/anas/raksha/Prefs.java" <<'RAKSHA_EOF'
          package com.anas.raksha;

          import android.content.Context;
          import android.content.SharedPreferences;

          import java.util.ArrayList;
          import java.util.List;

          final class Prefs {
              static final String NAME = "name";
              static final String PHONE = "phone";
              static final String AUTOCALL = "autocall";
              static final String GUARD = "guard";
              static final String SOS_ACTIVE = "sos_active";
              static final String FAKE_NAME = "fake_name";

              private Prefs() {}

              static SharedPreferences get(Context c) {
                  return c.getSharedPreferences("anas_raksha", Context.MODE_PRIVATE);
              }

              static List<String> phones(Context c) {
                  SharedPreferences p = get(c);
                  List<String> out = new ArrayList<>();
                  for (int i = 0; i < 5; i++) {
                      String n = p.getString(PHONE + i, "").replace(" ", "").trim();
                      if (!n.isEmpty()) out.add(n);
                  }
                  return out;
              }

              static String name(Context c) {
                  return get(c).getString(NAME, "").trim();
              }
          }
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/java/com/anas/raksha"
          cat > "raksha/app/src/main/java/com/anas/raksha/Util.java" <<'RAKSHA_EOF'
          package com.anas.raksha;

          import android.Manifest;
          import android.content.Context;
          import android.content.pm.PackageManager;
          import android.os.Build;
          import android.os.VibrationEffect;
          import android.os.Vibrator;
          import android.telephony.SmsManager;

          import java.util.ArrayList;

          final class Util {
              private Util() {}

              /** Sabhi trusted contacts ko SMS bhejta hai. Kitno ko gaya, wo return karta hai. */
              static int sendSms(Context c, String msg) {
                  if (c.checkSelfPermission(Manifest.permission.SEND_SMS) != PackageManager.PERMISSION_GRANTED) return 0;
                  SmsManager sm;
                  try {
                      if (Build.VERSION.SDK_INT >= 31) sm = c.getSystemService(SmsManager.class);
                      else sm = SmsManager.getDefault();
                  } catch (Exception e) {
                      return 0;
                  }
                  if (sm == null) return 0;
                  int ok = 0;
                  for (String num : Prefs.phones(c)) {
                      try {
                          ArrayList<String> parts = sm.divideMessage(msg);
                          if (parts.size() > 1) sm.sendMultipartTextMessage(num, null, parts, null, null);
                          else sm.sendTextMessage(num, null, msg, null, null);
                          ok++;
                      } catch (Exception ignored) {
                      }
                  }
                  return ok;
              }

              static void vibrate(Context c, long[] pattern, int repeat) {
                  try {
                      Vibrator v = (Vibrator) c.getSystemService(Context.VIBRATOR_SERVICE);
                      if (v == null || !v.hasVibrator()) return;
                      if (Build.VERSION.SDK_INT >= 26) v.vibrate(VibrationEffect.createWaveform(pattern, repeat));
                      else v.vibrate(pattern, repeat);
                  } catch (Exception ignored) {
                  }
              }

              static void stopVibrate(Context c) {
                  try {
                      Vibrator v = (Vibrator) c.getSystemService(Context.VIBRATOR_SERVICE);
                      if (v != null) v.cancel();
                  } catch (Exception ignored) {
                  }
              }
          }
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/java/com/anas/raksha"
          cat > "raksha/app/src/main/java/com/anas/raksha/Siren.java" <<'RAKSHA_EOF'
          package com.anas.raksha;

          import android.media.AudioFormat;
          import android.media.AudioManager;
          import android.media.AudioTrack;

          /** Tez police-jaisi siren aawaz banata hai (koi sound file nahi chahiye). */
          final class Siren {
              private volatile boolean running = false;

              void start() {
                  if (running) return;
                  running = true;
                  new Thread(() -> {
                      int sr = 22050;
                      int min = AudioTrack.getMinBufferSize(sr, AudioFormat.CHANNEL_OUT_MONO, AudioFormat.ENCODING_PCM_16BIT);
                      AudioTrack at;
                      try {
                          at = new AudioTrack(AudioManager.STREAM_MUSIC, sr, AudioFormat.CHANNEL_OUT_MONO,
                                  AudioFormat.ENCODING_PCM_16BIT, Math.max(min, 4096), AudioTrack.MODE_STREAM);
                          at.play();
                      } catch (Exception e) {
                          running = false;
                          return;
                      }
                      short[] buf = new short[1024];
                      double phase = 0, t = 0;
                      while (running) {
                          for (int i = 0; i < buf.length; i++) {
                              double f = 1150 + 550 * Math.sin(2 * Math.PI * t * 1.3);
                              phase += 2 * Math.PI * f / sr;
                              if (phase > 2 * Math.PI) phase -= 2 * Math.PI;
                              buf[i] = (short) (Math.sin(phase) > 0 ? 30000 : -30000);
                              t += 1.0 / sr;
                          }
                          at.write(buf, 0, buf.length);
                      }
                      try { at.stop(); } catch (Exception ignored) {}
                      at.release();
                  }).start();
              }

              void stop() {
                  running = false;
              }
          }
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/java/com/anas/raksha"
          cat > "raksha/app/src/main/java/com/anas/raksha/RakshaService.java" <<'RAKSHA_EOF'
          package com.anas.raksha;

          import android.Manifest;
          import android.app.Notification;
          import android.app.NotificationChannel;
          import android.app.NotificationManager;
          import android.app.PendingIntent;
          import android.app.Service;
          import android.content.BroadcastReceiver;
          import android.content.Context;
          import android.content.Intent;
          import android.content.IntentFilter;
          import android.content.pm.PackageManager;
          import android.content.pm.ServiceInfo;
          import android.hardware.Sensor;
          import android.hardware.SensorEvent;
          import android.hardware.SensorEventListener;
          import android.hardware.SensorManager;
          import android.location.Location;
          import android.location.LocationListener;
          import android.location.LocationManager;
          import android.os.Build;
          import android.os.Bundle;
          import android.os.Handler;
          import android.os.IBinder;
          import android.os.Looper;
          import android.os.PowerManager;
          import android.os.SystemClock;

          import java.util.ArrayList;
          import java.util.List;
          import java.util.Locale;

          /**
           * ANAS RAKSHA ka dil: SOS SMS + location updates + Guard mode
           * (power button 4 baar / phone hilana = SOS).
           */
          public class RakshaService extends Service implements SensorEventListener, LocationListener {

              static final String A_GUARD_ON = "com.anas.raksha.GUARD_ON";
              static final String A_GUARD_OFF = "com.anas.raksha.GUARD_OFF";
              static final String A_SOS = "com.anas.raksha.SOS";
              static final String A_SAFE = "com.anas.raksha.SAFE";
              static final String A_CANCEL = "com.anas.raksha.CANCEL";
              static final String ACTION_STATUS = "com.anas.raksha.STATUS";

              private static final String CH = "anas_raksha_alerts";
              private static final int NOTIF_ID = 7;
              private static final long UPDATE_EVERY = 2 * 60 * 1000L;   // har 2 minute
              private static final int MAX_UPDATES = 15;                 // 30 minute tak
              private static final long GRACE_MS = 6000;                 // auto-trigger cancel karne ka time

              private Handler h;
              private LocationManager lm;
              private SensorManager sm;
              private Location best;
              private boolean guard, sos, pending, firstSent, locListening, triggersOn;
              private int updates;
              private String reason = "SOS";
              private String statusLine = "";
              private long lastShake;
              private final List<Long> shakes = new ArrayList<>();
              private final List<Long> presses = new ArrayList<>();
              private PowerManager.WakeLock wl;

              private final BroadcastReceiver screenRx = new BroadcastReceiver() {
                  @Override
                  public void onReceive(Context c, Intent i) {
                      long now = SystemClock.elapsedRealtime();
                      presses.add(now);
                      while (!presses.isEmpty() && now - presses.get(0) > 4000) presses.remove(0);
                      if (presses.size() >= 4) {
                          presses.clear();
                          autoTrigger("Power button baar-baar dabaya");
                      }
                  }
              };

              private final Runnable firstSend = this::sendFirst;

              private final Runnable pendingFire = () -> {
                  pending = false;
                  startSos(reason);
              };

              private final Runnable periodic = new Runnable() {
                  @Override
                  public void run() {
                      if (!sos) return;
                      updates++;
                      String n = Prefs.name(RakshaService.this);
                      String who = n.isEmpty() ? "Meri" : n + " ki";
                      Util.sendSms(RakshaService.this, "SOS update " + updates + ": " + who + " nayi location\n" + loc() + "\n-ANAS RAKSHA");
                      statusLine = "Update " + updates + "/" + MAX_UPDATES + " bheja";
                      updateNotif();
                      if (updates < MAX_UPDATES) h.postDelayed(this, UPDATE_EVERY);
                      else stopLocation();
                  }
              };

              @Override
              public void onCreate() {
                  super.onCreate();
                  h = new Handler(Looper.getMainLooper());
                  lm = (LocationManager) getSystemService(LOCATION_SERVICE);
                  sm = (SensorManager) getSystemService(SENSOR_SERVICE);
                  if (Build.VERSION.SDK_INT >= 26) {
                      NotificationChannel ch = new NotificationChannel(CH, "ANAS RAKSHA", NotificationManager.IMPORTANCE_HIGH);
                      ch.setDescription("SOS aur Guard mode");
                      NotificationManager nm = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
                      if (nm != null) nm.createNotificationChannel(ch);
                  }
                  guard = Prefs.get(this).getBoolean(Prefs.GUARD, false);
                  if (guard) enableTriggers();
              }

              @Override
              public int onStartCommand(Intent intent, int flags, int startId) {
                  if (!goForeground()) {
                      stopSelf();
                      return START_NOT_STICKY;
                  }
                  String a = intent != null ? intent.getAction() : null;
                  if (a != null) {
                      switch (a) {
                          case A_GUARD_ON:
                              guard = true;
                              enableTriggers();
                              break;
                          case A_GUARD_OFF:
                              guard = false;
                              disableTriggers();
                              break;
                          case A_SOS:
                              String r = intent.getStringExtra("reason");
                              cancelPending();
                              startSos(r != null ? r : "SOS button dabaya");
                              break;
                          case A_SAFE:
                              cancelPending();
                              stopSos(true);
                              break;
                          case A_CANCEL:
                              cancelPending();
                              break;
                          default:
                              break;
                      }
                  }
                  persist();
                  updateNotif();
                  broadcast();
                  if (!guard && !sos && !pending) {
                      stopForeground(STOP_FOREGROUND_REMOVE);
                      stopSelf();
                      return START_NOT_STICKY;
                  }
                  return START_STICKY;
              }

              // ---------------- SOS ----------------

              private void startSos(String r) {
                  if (sos) return;
                  sos = true;
                  reason = r;
                  updates = 0;
                  firstSent = false;
                  statusLine = "Location dhoondh rahe hain...";
                  persist();
                  Util.vibrate(this, new long[]{0, 500, 200, 500, 200, 500}, -1);
                  acquireWakeLock();
                  best = lastKnown();
                  startLocation();
                  if (best != null && System.currentTimeMillis() - best.getTime() < 60_000) sendFirst();
                  else h.postDelayed(firstSend, 15_000);
                  updateNotif();
                  broadcast();
              }

              private void sendFirst() {
                  if (!sos || firstSent) return;
                  firstSent = true;
                  h.removeCallbacks(firstSend);
                  String n = Prefs.name(this);
                  String who = n.isEmpty() ? "Mujhe" : n + " ko";
                  String msg = "SOS! " + who + " turant madad chahiye!\n" + reason + "\n" + loc() + "\n-ANAS RAKSHA";
                  int sent = Util.sendSms(this, msg);
                  statusLine = sent > 0 ? sent + " contacts ko SMS gaya. Har 2 min update." : "SMS nahi gaya! 112 call karo.";
                  h.postDelayed(periodic, UPDATE_EVERY);
                  updateNotif();
                  broadcast();
              }

              private void stopSos(boolean sendSafe) {
                  if (!sos) return;
                  sos = false;
                  h.removeCallbacks(firstSend);
                  h.removeCallbacks(periodic);
                  stopLocation();
                  if (sendSafe) {
                      String n = Prefs.name(this);
                      Util.sendSms(this, (n.isEmpty() ? "Main" : n) + " ab SAFE hai. Fikar mat karo.\n-ANAS RAKSHA");
                  }
                  statusLine = "";
                  releaseWakeLock();
                  persist();
              }

              private void autoTrigger(String r) {
                  if (sos || pending) return;
                  pending = true;
                  reason = r;
                  Util.vibrate(this, new long[]{0, 1000, 300, 1000, 300, 1000}, -1);
                  h.postDelayed(pendingFire, GRACE_MS);
                  updateNotif();
                  broadcast();
              }

              private void cancelPending() {
                  if (!pending) return;
                  pending = false;
                  h.removeCallbacks(pendingFire);
                  Util.stopVibrate(this);
                  updateNotif();
              }

              private String loc() {
                  if (best == null) return "Location nahi mili (GPS on karo)";
                  return String.format(Locale.US, "Location: https://maps.google.com/?q=%.6f,%.6f",
                          best.getLatitude(), best.getLongitude());
              }

              // ---------------- Location ----------------

              private boolean hasLocPerm() {
                  return checkSelfPermission(Manifest.permission.ACCESS_FINE_LOCATION) == PackageManager.PERMISSION_GRANTED
                          || checkSelfPermission(Manifest.permission.ACCESS_COARSE_LOCATION) == PackageManager.PERMISSION_GRANTED;
              }

              private Location lastKnown() {
                  if (lm == null || !hasLocPerm()) return null;
                  Location out = null;
                  for (String p : new String[]{LocationManager.GPS_PROVIDER, LocationManager.NETWORK_PROVIDER}) {
                      try {
                          Location l = lm.getLastKnownLocation(p);
                          if (l != null && (out == null || l.getTime() > out.getTime())) out = l;
                      } catch (Exception ignored) {
                      }
                  }
                  return out;
              }

              private void startLocation() {
                  if (locListening || lm == null || !hasLocPerm()) return;
                  for (String p : new String[]{LocationManager.GPS_PROVIDER, LocationManager.NETWORK_PROVIDER}) {
                      try {
                          if (lm.isProviderEnabled(p)) {
                              lm.requestLocationUpdates(p, 5000, 5, this, Looper.getMainLooper());
                              locListening = true;
                          }
                      } catch (Exception ignored) {
                      }
                  }
              }

              private void stopLocation() {
                  try {
                      if (lm != null) lm.removeUpdates(this);
                  } catch (Exception ignored) {
                  }
                  locListening = false;
              }

              @Override
              public void onLocationChanged(Location l) {
                  if (l == null) return;
                  if (best == null || l.getAccuracy() <= best.getAccuracy() + 20 || l.getTime() - best.getTime() > 30_000) {
                      best = l;
                  }
                  if (sos && !firstSent) sendFirst();
              }

              @Override public void onStatusChanged(String provider, int status, Bundle extras) {}
              @Override public void onProviderEnabled(String provider) {}
              @Override public void onProviderDisabled(String provider) {}

              // ---------------- Guard triggers ----------------

              private void enableTriggers() {
                  if (triggersOn) return;
                  triggersOn = true;
                  IntentFilter f = new IntentFilter();
                  f.addAction(Intent.ACTION_SCREEN_ON);
                  f.addAction(Intent.ACTION_SCREEN_OFF);
                  try {
                      if (Build.VERSION.SDK_INT >= 33) registerReceiver(screenRx, f, Context.RECEIVER_NOT_EXPORTED);
                      else registerReceiver(screenRx, f);
                  } catch (Exception ignored) {
                  }
                  if (sm != null) {
                      Sensor acc = sm.getDefaultSensor(Sensor.TYPE_ACCELEROMETER);
                      if (acc != null) sm.registerListener(this, acc, SensorManager.SENSOR_DELAY_GAME);
                  }
              }

              private void disableTriggers() {
                  if (!triggersOn) return;
                  triggersOn = false;
                  try {
                      unregisterReceiver(screenRx);
                  } catch (Exception ignored) {
                  }
                  if (sm != null) sm.unregisterListener(this);
              }

              @Override
              public void onSensorChanged(SensorEvent e) {
                  float x = e.values[0], y = e.values[1], z = e.values[2];
                  double g = Math.sqrt(x * x + y * y + z * z) / SensorManager.GRAVITY_EARTH;
                  if (g > 2.7) {
                      long now = SystemClock.elapsedRealtime();
                      if (now - lastShake > 250) {
                          lastShake = now;
                          shakes.add(now);
                      }
                      while (!shakes.isEmpty() && now - shakes.get(0) > 2500) shakes.remove(0);
                      if (shakes.size() >= 4) {
                          shakes.clear();
                          autoTrigger("Phone zor se hilaya (shake)");
                      }
                  }
              }

              @Override public void onAccuracyChanged(Sensor sensor, int accuracy) {}

              // ---------------- Notification & state ----------------

              private boolean goForeground() {
                  try {
                      Notification n = buildNotif();
                      if (Build.VERSION.SDK_INT >= 29) startForeground(NOTIF_ID, n, ServiceInfo.FOREGROUND_SERVICE_TYPE_LOCATION);
                      else startForeground(NOTIF_ID, n);
                      return true;
                  } catch (Exception e) {
                      return false;
                  }
              }

              private PendingIntent svc(String action) {
                  Intent i = new Intent(this, RakshaService.class).setAction(action);
                  return PendingIntent.getService(this, action.hashCode(), i,
                          PendingIntent.FLAG_IMMUTABLE | PendingIntent.FLAG_UPDATE_CURRENT);
              }

              @SuppressWarnings("deprecation")
              private Notification buildNotif() {
                  Notification.Builder b = Build.VERSION.SDK_INT >= 26
                          ? new Notification.Builder(this, CH) : new Notification.Builder(this);
                  PendingIntent open = PendingIntent.getActivity(this, 0, new Intent(this, MainActivity.class),
                          PendingIntent.FLAG_IMMUTABLE | PendingIntent.FLAG_UPDATE_CURRENT);
                  b.setSmallIcon(R.drawable.ic_shield).setContentIntent(open).setOngoing(true).setOnlyAlertOnce(true);
                  if (pending) {
                      b.setContentTitle("⚠️ 6 second mein SOS jaayega!")
                              .setContentText(reason + " - galti se hua to CANCEL dabao")
                              .addAction(0, "❌ CANCEL", svc(A_CANCEL));
                  } else if (sos) {
                      b.setContentTitle("🚨 SOS CHALU HAI")
                              .setContentText(statusLine.isEmpty() ? "Contacts ko location SMS ja rahi hai" : statusLine)
                              .addAction(0, "✅ Main safe hoon", svc(A_SAFE));
                  } else if (guard) {
                      b.setContentTitle("🛡️ ANAS RAKSHA Guard ON")
                              .setContentText("Power button 4 baar dabao ya phone hilao = SOS")
                              .addAction(0, "Guard band karo", svc(A_GUARD_OFF));
                  } else {
                      b.setContentTitle("ANAS RAKSHA").setContentText("Made by Anas");
                  }
                  return b.build();
              }

              private void updateNotif() {
                  try {
                      NotificationManager nm = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
                      if (nm != null) nm.notify(NOTIF_ID, buildNotif());
                  } catch (Exception ignored) {
                  }
              }

              private void persist() {
                  Prefs.get(this).edit().putBoolean(Prefs.GUARD, guard).putBoolean(Prefs.SOS_ACTIVE, sos).apply();
              }

              private void broadcast() {
                  sendBroadcast(new Intent(ACTION_STATUS).setPackage(getPackageName()));
              }

              private void acquireWakeLock() {
                  try {
                      PowerManager pm = (PowerManager) getSystemService(POWER_SERVICE);
                      if (pm == null) return;
                      if (wl == null) wl = pm.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "anasraksha:sos");
                      if (!wl.isHeld()) wl.acquire(40 * 60 * 1000L);
                  } catch (Exception ignored) {
                  }
              }

              private void releaseWakeLock() {
                  try {
                      if (wl != null && wl.isHeld()) wl.release();
                  } catch (Exception ignored) {
                  }
              }

              @Override
              public void onDestroy() {
                  disableTriggers();
                  stopLocation();
                  h.removeCallbacksAndMessages(null);
                  releaseWakeLock();
                  sos = false;
                  pending = false;
                  Prefs.get(this).edit().putBoolean(Prefs.SOS_ACTIVE, false).apply();
                  broadcast();
                  super.onDestroy();
              }

              @Override
              public IBinder onBind(Intent intent) {
                  return null;
              }
          }
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/java/com/anas/raksha"
          cat > "raksha/app/src/main/java/com/anas/raksha/MainActivity.java" <<'RAKSHA_EOF'
          package com.anas.raksha;

          import android.Manifest;
          import android.app.Activity;
          import android.app.AlertDialog;
          import android.content.BroadcastReceiver;
          import android.content.Context;
          import android.content.Intent;
          import android.content.IntentFilter;
          import android.content.SharedPreferences;
          import android.content.pm.PackageManager;
          import android.hardware.camera2.CameraCharacteristics;
          import android.hardware.camera2.CameraManager;
          import android.media.AudioManager;
          import android.net.Uri;
          import android.os.Build;
          import android.os.Bundle;
          import android.os.CountDownTimer;
          import android.os.Handler;
          import android.os.Looper;
          import android.view.View;
          import android.widget.Button;
          import android.widget.CheckBox;
          import android.widget.EditText;
          import android.widget.TextView;
          import android.widget.Toast;

          import java.util.ArrayList;
          import java.util.List;

          /** ANAS RAKSHA - Made by Anas */
          public class MainActivity extends Activity {

              private TextView statusTv;
              private Button safeBtn, sirenBtn, guardBtn;
              private EditText nameEt, fakeNameEt;
              private CheckBox autoCallCb;
              private final EditText[] phoneEts = new EditText[5];
              private final Handler handler = new Handler(Looper.getMainLooper());

              private AlertDialog cdDialog;
              private CountDownTimer cdTimer;

              private final Siren siren = new Siren();
              private boolean sirenOn = false;
              private int oldVolume = -1;
              private CameraManager camMgr;
              private String torchId;
              private boolean torchState = false;

              private final Runnable torchBlink = new Runnable() {
                  @Override
                  public void run() {
                      if (!sirenOn) {
                          setTorch(false);
                          return;
                      }
                      setTorch(!torchState);
                      handler.postDelayed(this, 250);
                  }
              };

              private final BroadcastReceiver statusRx = new BroadcastReceiver() {
                  @Override
                  public void onReceive(Context c, Intent i) {
                      refresh();
                  }
              };

              @Override
              protected void onCreate(Bundle savedInstanceState) {
                  super.onCreate(savedInstanceState);
                  setContentView(R.layout.activity_main);

                  statusTv = findViewById(R.id.status);
                  safeBtn = findViewById(R.id.safeBtn);
                  sirenBtn = findViewById(R.id.sirenBtn);
                  guardBtn = findViewById(R.id.guardBtn);
                  nameEt = findViewById(R.id.nameEt);
                  fakeNameEt = findViewById(R.id.fakeNameEt);
                  autoCallCb = findViewById(R.id.autoCallCb);
                  int[] ids = {R.id.phone1, R.id.phone2, R.id.phone3, R.id.phone4, R.id.phone5};
                  for (int i = 0; i < 5; i++) phoneEts[i] = findViewById(ids[i]);

                  SharedPreferences p = Prefs.get(this);
                  nameEt.setText(p.getString(Prefs.NAME, ""));
                  for (int i = 0; i < 5; i++) phoneEts[i].setText(p.getString(Prefs.PHONE + i, ""));
                  autoCallCb.setChecked(p.getBoolean(Prefs.AUTOCALL, true));
                  fakeNameEt.setText(p.getString(Prefs.FAKE_NAME, "Papa"));

                  findViewById(R.id.sosBtn).setOnClickListener(v -> startCountdown());
                  safeBtn.setOnClickListener(v -> confirmSafe());
                  sirenBtn.setOnClickListener(v -> toggleSiren());
                  guardBtn.setOnClickListener(v -> toggleGuard());
                  findViewById(R.id.fakeCallBtn).setOnClickListener(v -> scheduleFakeCall());
                  findViewById(R.id.saveBtn).setOnClickListener(v -> {
                      save();
                      toast("Save ho gaya ✓");
                  });
                  findViewById(R.id.testBtn).setOnClickListener(v -> testSms());
                  findViewById(R.id.call112).setOnClickListener(v -> call("112"));
                  findViewById(R.id.call1091).setOnClickListener(v -> call("1091"));
                  findViewById(R.id.call181).setOnClickListener(v -> call("181"));
                  findViewById(R.id.callNcw).setOnClickListener(v -> call("7827170170"));

                  findTorch();
                  askPermissions();
              }

              @Override
              protected void onResume() {
                  super.onResume();
                  IntentFilter f = new IntentFilter(RakshaService.ACTION_STATUS);
                  if (Build.VERSION.SDK_INT >= 33) registerReceiver(statusRx, f, Context.RECEIVER_NOT_EXPORTED);
                  else registerReceiver(statusRx, f);
                  refresh();
              }

              @Override
              protected void onPause() {
                  super.onPause();
                  try {
                      unregisterReceiver(statusRx);
                  } catch (Exception ignored) {
                  }
              }

              @Override
              protected void onDestroy() {
                  if (sirenOn) toggleSiren();
                  super.onDestroy();
              }

              // ---------------- UI state ----------------

              private void refresh() {
                  SharedPreferences p = Prefs.get(this);
                  boolean sos = p.getBoolean(Prefs.SOS_ACTIVE, false);
                  boolean guard = p.getBoolean(Prefs.GUARD, false);
                  int n = Prefs.phones(this).size();

                  safeBtn.setVisibility(sos ? View.VISIBLE : View.GONE);
                  guardBtn.setText(guard
                          ? "🛡️ Guard Mode ON\nPower button 4x ya shake = SOS  (band karne ko dabao)"
                          : "🛡️ Guard Mode OFF\nChalu karo: power button 4x / shake se SOS");
                  guardBtn.setBackgroundResource(guard ? R.drawable.btn_green : R.drawable.btn_bg);

                  if (sos) {
                      statusTv.setText("🚨 SOS CHALU HAI\nContacts ko har 2 minute location SMS ja rahi hai.\nSafe ho to neeche wala button dabao.");
                  } else if (n == 0) {
                      statusTv.setText("⚠️ Pehle neeche apne trusted contacts ke number daalo aur Save karo.");
                  } else {
                      statusTv.setText("Khatre mein ho? SOS dabao.\n" + n + " contacts ko SMS + location apne aap jaayegi.");
                  }
              }

              private void save() {
                  SharedPreferences.Editor e = Prefs.get(this).edit();
                  e.putString(Prefs.NAME, nameEt.getText().toString().trim());
                  for (int i = 0; i < 5; i++) e.putString(Prefs.PHONE + i, phoneEts[i].getText().toString().trim());
                  e.putBoolean(Prefs.AUTOCALL, autoCallCb.isChecked());
                  e.putString(Prefs.FAKE_NAME, fakeNameEt.getText().toString().trim());
                  e.apply();
                  refresh();
              }

              // ---------------- Permissions ----------------

              private boolean granted(String perm) {
                  return checkSelfPermission(perm) == PackageManager.PERMISSION_GRANTED;
              }

              private boolean hasCorePerms() {
                  return granted(Manifest.permission.SEND_SMS) && granted(Manifest.permission.ACCESS_FINE_LOCATION);
              }

              private void askPermissions() {
                  List<String> need = new ArrayList<>();
                  String[] all = {Manifest.permission.SEND_SMS, Manifest.permission.ACCESS_FINE_LOCATION,
                          Manifest.permission.ACCESS_COARSE_LOCATION, Manifest.permission.CALL_PHONE};
                  for (String s : all) if (!granted(s)) need.add(s);
                  if (Build.VERSION.SDK_INT >= 33 && !granted(Manifest.permission.POST_NOTIFICATIONS)) {
                      need.add(Manifest.permission.POST_NOTIFICATIONS);
                  }
                  if (!need.isEmpty()) requestPermissions(need.toArray(new String[0]), 1);
              }

              private boolean readyCheck() {
                  save();
                  if (Prefs.phones(this).isEmpty()) {
                      toast("Pehle kam se kam 1 contact number daalo");
                      return false;
                  }
                  if (!hasCorePerms()) {
                      askPermissions();
                      toast("SMS aur Location permission do, phir dobara try karo");
                      return false;
                  }
                  return true;
              }

              // ---------------- SOS ----------------

              private void startCountdown() {
                  if (!readyCheck()) return;
                  if (Prefs.get(this).getBoolean(Prefs.SOS_ACTIVE, false)) {
                      toast("SOS pehle se chalu hai");
                      return;
                  }
                  cdDialog = new AlertDialog.Builder(this)
                          .setTitle("🚨 SOS bhej rahe hain...")
                          .setMessage("3")
                          .setCancelable(false)
                          .setNegativeButton("❌ Galti se dab gaya - Cancel", (d, w) -> {
                              if (cdTimer != null) cdTimer.cancel();
                              toast("Cancel ho gaya");
                          })
                          .create();
                  cdDialog.show();
                  cdTimer = new CountDownTimer(3000, 1000) {
                      @Override
                      public void onTick(long ms) {
                          cdDialog.setMessage(String.valueOf((ms + 999) / 1000));
                          Util.vibrate(MainActivity.this, new long[]{0, 120}, -1);
                      }

                      @Override
                      public void onFinish() {
                          if (cdDialog.isShowing()) cdDialog.dismiss();
                          fireSos();
                      }
                  }.start();
              }

              private void fireSos() {
                  startSvc(new Intent(this, RakshaService.class)
                          .setAction(RakshaService.A_SOS)
                          .putExtra("reason", "SOS button dabaya"));
                  if (autoCallCb.isChecked()) {
                      List<String> ph = Prefs.phones(this);
                      if (!ph.isEmpty()) {
                          final String first = ph.get(0);
                          handler.postDelayed(() -> call(first), 5000);
                      }
                  }
                  handler.postDelayed(this::refresh, 600);
              }

              private void confirmSafe() {
                  new AlertDialog.Builder(this)
                          .setTitle("Pakka safe ho?")
                          .setMessage("Sabhi contacts ko \"safe hoon\" SMS jaayega aur location updates band ho jaayenge.")
                          .setPositiveButton("✅ Haan, safe hoon", (d, w) -> {
                              startSvc(new Intent(this, RakshaService.class).setAction(RakshaService.A_SAFE));
                              if (sirenOn) toggleSiren();
                              handler.postDelayed(this::refresh, 600);
                          })
                          .setNegativeButton("Nahi", null)
                          .show();
              }

              private void toggleGuard() {
                  boolean on = Prefs.get(this).getBoolean(Prefs.GUARD, false);
                  if (on) {
                      startSvc(new Intent(this, RakshaService.class).setAction(RakshaService.A_GUARD_OFF));
                      toast("Guard Mode band");
                  } else {
                      if (!readyCheck()) return;
                      startSvc(new Intent(this, RakshaService.class).setAction(RakshaService.A_GUARD_ON));
                      toast("Guard Mode ON 🛡️");
                  }
                  handler.postDelayed(this::refresh, 600);
              }

              private void startSvc(Intent i) {
                  try {
                      if (Build.VERSION.SDK_INT >= 26) startForegroundService(i);
                      else startService(i);
                  } catch (Exception e) {
                      toast("Service start nahi hui: " + e.getMessage());
                  }
              }

              private void testSms() {
                  if (!readyCheck()) return;
                  String n = Prefs.name(this);
                  String msg = "ANAS RAKSHA test: " + (n.isEmpty() ? "Maine" : n + " ne")
                          + " aapko emergency contact banaya hai. Khatre ke time aapko yahin SMS mein location milegi.";
                  int sent = Util.sendSms(this, msg);
                  toast(sent + " logon ko test SMS bheja");
              }

              private void call(String num) {
                  Intent i;
                  if (granted(Manifest.permission.CALL_PHONE)) i = new Intent(Intent.ACTION_CALL, Uri.parse("tel:" + num));
                  else i = new Intent(Intent.ACTION_DIAL, Uri.parse("tel:" + num));
                  try {
                      startActivity(i);
                  } catch (Exception e) {
                      toast("Call nahi ho paayi");
                  }
              }

              // ---------------- Siren + flashlight ----------------

              private void toggleSiren() {
                  AudioManager am = (AudioManager) getSystemService(AUDIO_SERVICE);
                  sirenOn = !sirenOn;
                  if (sirenOn) {
                      if (am != null) {
                          oldVolume = am.getStreamVolume(AudioManager.STREAM_MUSIC);
                          am.setStreamVolume(AudioManager.STREAM_MUSIC, am.getStreamMaxVolume(AudioManager.STREAM_MUSIC), 0);
                      }
                      siren.start();
                      handler.post(torchBlink);
                      Util.vibrate(this, new long[]{0, 500, 200, 500}, -1);
                      sirenBtn.setText("🔇 Siren band karo");
                      sirenBtn.setBackgroundResource(R.drawable.btn_red);
                  } else {
                      siren.stop();
                      setTorch(false);
                      if (am != null && oldVolume >= 0) am.setStreamVolume(AudioManager.STREAM_MUSIC, oldVolume, 0);
                      sirenBtn.setText("📢 Siren");
                      sirenBtn.setBackgroundResource(R.drawable.btn_bg);
                  }
              }

              private void findTorch() {
                  try {
                      camMgr = (CameraManager) getSystemService(CAMERA_SERVICE);
                      if (camMgr == null) return;
                      for (String id : camMgr.getCameraIdList()) {
                          Boolean f = camMgr.getCameraCharacteristics(id).get(CameraCharacteristics.FLASH_INFO_AVAILABLE);
                          if (Boolean.TRUE.equals(f)) {
                              torchId = id;
                              break;
                          }
                      }
                  } catch (Exception ignored) {
                  }
              }

              private void setTorch(boolean on) {
                  if (camMgr == null || torchId == null) return;
                  try {
                      camMgr.setTorchMode(torchId, on);
                      torchState = on;
                  } catch (Exception ignored) {
                  }
              }

              // ---------------- Fake call ----------------

              private void scheduleFakeCall() {
                  save();
                  toast("10 second mein fake call aayegi... app khula rakho");
                  final String name = fakeNameEt.getText().toString().trim();
                  handler.postDelayed(() -> startActivity(new Intent(this, FakeCallActivity.class)
                          .putExtra("name", name.isEmpty() ? "Papa" : name)), 10_000);
              }

              private void toast(String s) {
                  Toast.makeText(this, s, Toast.LENGTH_SHORT).show();
              }
          }
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/java/com/anas/raksha"
          cat > "raksha/app/src/main/java/com/anas/raksha/FakeCallActivity.java" <<'RAKSHA_EOF'
          package com.anas.raksha;

          import android.app.Activity;
          import android.media.Ringtone;
          import android.media.RingtoneManager;
          import android.os.Build;
          import android.os.Bundle;
          import android.os.Handler;
          import android.os.Looper;
          import android.view.View;
          import android.view.WindowManager;
          import android.widget.TextView;

          import java.util.Locale;

          /** Nakli incoming call screen - awkward situation se nikalne ke liye. */
          public class FakeCallActivity extends Activity {

              private Ringtone rt;
              private final Handler h = new Handler(Looper.getMainLooper());
              private int secs = 0;
              private TextView sub;

              private final Runnable tick = new Runnable() {
                  @Override
                  public void run() {
                      secs++;
                      sub.setText(String.format(Locale.US, "%02d:%02d", secs / 60, secs % 60));
                      h.postDelayed(this, 1000);
                  }
              };

              @Override
              protected void onCreate(Bundle savedInstanceState) {
                  super.onCreate(savedInstanceState);
                  if (Build.VERSION.SDK_INT >= 27) {
                      setShowWhenLocked(true);
                      setTurnScreenOn(true);
                  } else {
                      getWindow().addFlags(WindowManager.LayoutParams.FLAG_SHOW_WHEN_LOCKED
                              | WindowManager.LayoutParams.FLAG_TURN_SCREEN_ON);
                  }
                  getWindow().addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON);
                  setContentView(R.layout.activity_fake_call);

                  String name = getIntent().getStringExtra("name");
                  ((TextView) findViewById(R.id.fcName)).setText(name == null ? "Papa" : name);
                  sub = findViewById(R.id.fcSub);

                  try {
                      rt = RingtoneManager.getRingtone(this, RingtoneManager.getDefaultUri(RingtoneManager.TYPE_RINGTONE));
                      if (rt != null) rt.play();
                  } catch (Exception ignored) {
                  }
                  Util.vibrate(this, new long[]{0, 900, 700}, 0);

                  findViewById(R.id.fcDecline).setOnClickListener(v -> finish());
                  findViewById(R.id.fcEnd).setOnClickListener(v -> finish());
                  findViewById(R.id.fcAccept).setOnClickListener(v -> {
                      stopRing();
                      findViewById(R.id.fcActions).setVisibility(View.GONE);
                      findViewById(R.id.fcEnd).setVisibility(View.VISIBLE);
                      sub.setText("00:00");
                      h.postDelayed(tick, 1000);
                  });
              }

              private void stopRing() {
                  try {
                      if (rt != null) rt.stop();
                  } catch (Exception ignored) {
                  }
                  Util.stopVibrate(this);
              }

              @Override
              protected void onDestroy() {
                  stopRing();
                  h.removeCallbacks(tick);
                  super.onDestroy();
              }
          }
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/res/values"
          cat > "raksha/app/src/main/res/values/strings.xml" <<'RAKSHA_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <resources>
              <string name="app_name">ANAS RAKSHA</string>
          </resources>
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/res/values"
          cat > "raksha/app/src/main/res/values/colors.xml" <<'RAKSHA_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <resources>
              <color name="bg">#0F0F14</color>
              <color name="text">#F2F2F5</color>
              <color name="muted">#9A9AA8</color>
              <color name="pink">#FF6B8B</color>
          </resources>
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/res/values"
          cat > "raksha/app/src/main/res/values/styles.xml" <<'RAKSHA_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <resources>
              <style name="AppTheme" parent="android:Theme.Material.NoActionBar">
                  <item name="android:windowBackground">@color/bg</item>
                  <item name="android:statusBarColor">@color/bg</item>
                  <item name="android:navigationBarColor">@color/bg</item>
                  <item name="android:colorAccent">@color/pink</item>
              </style>

              <style name="Btn">
                  <item name="android:layout_width">match_parent</item>
                  <item name="android:layout_height">wrap_content</item>
                  <item name="android:layout_marginTop">10dp</item>
                  <item name="android:minHeight">54dp</item>
                  <item name="android:background">@drawable/btn_bg</item>
                  <item name="android:textColor">@color/text</item>
                  <item name="android:textSize">15sp</item>
                  <item name="android:textStyle">bold</item>
                  <item name="android:textAllCaps">false</item>
                  <item name="android:stateListAnimator">@null</item>
                  <item name="android:padding">12dp</item>
              </style>

              <style name="HalfBtn" parent="Btn">
                  <item name="android:layout_width">0dp</item>
                  <item name="android:layout_weight">1</item>
              </style>

              <style name="Field">
                  <item name="android:layout_width">match_parent</item>
                  <item name="android:layout_height">wrap_content</item>
                  <item name="android:layout_marginTop">8dp</item>
                  <item name="android:background">@drawable/field_bg</item>
                  <item name="android:textColor">@color/text</item>
                  <item name="android:textColorHint">#77778A</item>
                  <item name="android:textSize">15sp</item>
                  <item name="android:padding">14dp</item>
                  <item name="android:singleLine">true</item>
              </style>

              <style name="Section">
                  <item name="android:layout_width">match_parent</item>
                  <item name="android:layout_height">wrap_content</item>
                  <item name="android:layout_marginTop">26dp</item>
                  <item name="android:textColor">@color/text</item>
                  <item name="android:textSize">16sp</item>
                  <item name="android:textStyle">bold</item>
              </style>
          </resources>
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/res/layout"
          cat > "raksha/app/src/main/res/layout/activity_main.xml" <<'RAKSHA_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
              android:layout_width="match_parent"
              android:layout_height="match_parent"
              android:background="@color/bg"
              android:fillViewport="true">

              <LinearLayout
                  android:layout_width="match_parent"
                  android:layout_height="wrap_content"
                  android:orientation="vertical"
                  android:padding="18dp">

                  <LinearLayout
                      android:layout_width="wrap_content"
                      android:layout_height="wrap_content"
                      android:orientation="horizontal">
                      <TextView
                          android:layout_width="wrap_content"
                          android:layout_height="wrap_content"
                          android:text="ANAS "
                          android:textColor="@color/text"
                          android:textSize="26sp"
                          android:textStyle="bold" />
                      <TextView
                          android:layout_width="wrap_content"
                          android:layout_height="wrap_content"
                          android:text="RAKSHA"
                          android:textColor="@color/pink"
                          android:textSize="26sp"
                          android:textStyle="bold" />
                  </LinearLayout>

                  <TextView
                      android:layout_width="wrap_content"
                      android:layout_height="wrap_content"
                      android:text="Har kadam pe saath · Made by Anas"
                      android:textColor="@color/muted"
                      android:textSize="12sp" />

                  <Button
                      android:id="@+id/sosBtn"
                      android:layout_width="220dp"
                      android:layout_height="220dp"
                      android:layout_gravity="center_horizontal"
                      android:layout_marginTop="28dp"
                      android:background="@drawable/sos_bg"
                      android:stateListAnimator="@null"
                      android:text="SOS"
                      android:textAllCaps="false"
                      android:textColor="#FFFFFF"
                      android:textSize="54sp"
                      android:textStyle="bold" />

                  <TextView
                      android:layout_width="match_parent"
                      android:layout_height="wrap_content"
                      android:layout_marginTop="8dp"
                      android:gravity="center"
                      android:text="Dabao - 3 second mein alert jaayega"
                      android:textColor="@color/muted"
                      android:textSize="12sp" />

                  <TextView
                      android:id="@+id/status"
                      android:layout_width="match_parent"
                      android:layout_height="wrap_content"
                      android:layout_marginTop="14dp"
                      android:gravity="center"
                      android:lineSpacingExtra="3dp"
                      android:textColor="#CFCFD8"
                      android:textSize="14sp" />

                  <Button
                      android:id="@+id/safeBtn"
                      style="@style/Btn"
                      android:background="@drawable/btn_green"
                      android:text="✅ Main safe hoon - alert band karo"
                      android:visibility="gone" />

                  <LinearLayout
                      android:layout_width="match_parent"
                      android:layout_height="wrap_content"
                      android:layout_marginTop="8dp"
                      android:orientation="horizontal">
                      <Button
                          android:id="@+id/sirenBtn"
                          style="@style/HalfBtn"
                          android:layout_marginEnd="5dp"
                          android:text="📢 Siren" />
                      <Button
                          android:id="@+id/fakeCallBtn"
                          style="@style/HalfBtn"
                          android:layout_marginStart="5dp"
                          android:text="📞 Fake Call" />
                  </LinearLayout>

                  <Button
                      android:id="@+id/guardBtn"
                      style="@style/Btn"
                      android:text="Guard Mode" />

                  <TextView
                      style="@style/Section"
                      android:text="☎️ Helpline (India)" />

                  <LinearLayout
                      android:layout_width="match_parent"
                      android:layout_height="wrap_content"
                      android:orientation="horizontal">
                      <Button
                          android:id="@+id/call112"
                          style="@style/HalfBtn"
                          android:layout_marginEnd="5dp"
                          android:text="112\nEmergency" />
                      <Button
                          android:id="@+id/call1091"
                          style="@style/HalfBtn"
                          android:layout_marginStart="5dp"
                          android:text="1091\nWomen Helpline" />
                  </LinearLayout>

                  <LinearLayout
                      android:layout_width="match_parent"
                      android:layout_height="wrap_content"
                      android:orientation="horizontal">
                      <Button
                          android:id="@+id/call181"
                          style="@style/HalfBtn"
                          android:layout_marginEnd="5dp"
                          android:text="181\nWomen Helpline" />
                      <Button
                          android:id="@+id/callNcw"
                          style="@style/HalfBtn"
                          android:layout_marginStart="5dp"
                          android:text="NCW\n7827170170" />
                  </LinearLayout>

                  <TextView
                      style="@style/Section"
                      android:text="👥 Trusted contacts (inko SMS jaayega)" />

                  <EditText
                      android:id="@+id/nameEt"
                      style="@style/Field"
                      android:hint="Aapka naam (jaise Ayesha)"
                      android:inputType="textPersonName" />
                  <EditText
                      android:id="@+id/phone1"
                      style="@style/Field"
                      android:hint="Contact 1 ka number"
                      android:inputType="phone" />
                  <EditText
                      android:id="@+id/phone2"
                      style="@style/Field"
                      android:hint="Contact 2 ka number"
                      android:inputType="phone" />
                  <EditText
                      android:id="@+id/phone3"
                      style="@style/Field"
                      android:hint="Contact 3 ka number"
                      android:inputType="phone" />
                  <EditText
                      android:id="@+id/phone4"
                      style="@style/Field"
                      android:hint="Contact 4 ka number"
                      android:inputType="phone" />
                  <EditText
                      android:id="@+id/phone5"
                      style="@style/Field"
                      android:hint="Contact 5 ka number"
                      android:inputType="phone" />

                  <CheckBox
                      android:id="@+id/autoCallCb"
                      android:layout_width="match_parent"
                      android:layout_height="wrap_content"
                      android:layout_marginTop="10dp"
                      android:buttonTint="@color/pink"
                      android:text="SOS ke baad Contact 1 ko call bhi lagao"
                      android:textColor="@color/text"
                      android:textSize="14sp" />

                  <Button
                      android:id="@+id/saveBtn"
                      style="@style/Btn"
                      android:background="@drawable/btn_pink"
                      android:text="💾 Save karo"
                      android:textColor="#1A0A10" />
                  <Button
                      android:id="@+id/testBtn"
                      style="@style/Btn"
                      android:text="🧪 Test SMS bhejo" />

                  <TextView
                      style="@style/Section"
                      android:text="📞 Fake call kiske naam se aaye?" />
                  <EditText
                      android:id="@+id/fakeNameEt"
                      style="@style/Field"
                      android:hint="Papa"
                      android:inputType="textPersonName" />

                  <TextView
                      android:layout_width="match_parent"
                      android:layout_height="wrap_content"
                      android:layout_marginTop="30dp"
                      android:layout_marginBottom="10dp"
                      android:gravity="center"
                      android:text="ANAS RAKSHA · Made with ❤️ by Anas"
                      android:textColor="@color/muted"
                      android:textSize="12sp" />
              </LinearLayout>
          </ScrollView>
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/res/layout"
          cat > "raksha/app/src/main/res/layout/activity_fake_call.xml" <<'RAKSHA_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
              android:layout_width="match_parent"
              android:layout_height="match_parent"
              android:background="#13202D"
              android:gravity="center_horizontal"
              android:orientation="vertical"
              android:paddingTop="110dp">

              <TextView
                  android:layout_width="110dp"
                  android:layout_height="110dp"
                  android:background="@drawable/circle_avatar"
                  android:gravity="center"
                  android:text="👤"
                  android:textSize="52sp" />

              <TextView
                  android:id="@+id/fcName"
                  android:layout_width="wrap_content"
                  android:layout_height="wrap_content"
                  android:layout_marginTop="18dp"
                  android:textColor="#FFFFFF"
                  android:textSize="32sp" />

              <TextView
                  android:id="@+id/fcSub"
                  android:layout_width="wrap_content"
                  android:layout_height="wrap_content"
                  android:layout_marginTop="6dp"
                  android:text="Incoming call..."
                  android:textColor="#9FB2C6"
                  android:textSize="16sp" />

              <View
                  android:layout_width="1dp"
                  android:layout_height="0dp"
                  android:layout_weight="1" />

              <LinearLayout
                  android:id="@+id/fcActions"
                  android:layout_width="wrap_content"
                  android:layout_height="wrap_content"
                  android:layout_marginBottom="90dp"
                  android:orientation="horizontal">
                  <Button
                      android:id="@+id/fcDecline"
                      android:layout_width="74dp"
                      android:layout_height="74dp"
                      android:layout_marginEnd="70dp"
                      android:background="@drawable/circle_red"
                      android:stateListAnimator="@null"
                      android:text="✕"
                      android:textColor="#FFFFFF"
                      android:textSize="28sp" />
                  <Button
                      android:id="@+id/fcAccept"
                      android:layout_width="74dp"
                      android:layout_height="74dp"
                      android:background="@drawable/circle_green"
                      android:stateListAnimator="@null"
                      android:text="✆"
                      android:textColor="#FFFFFF"
                      android:textSize="30sp" />
              </LinearLayout>

              <Button
                  android:id="@+id/fcEnd"
                  android:layout_width="74dp"
                  android:layout_height="74dp"
                  android:layout_marginBottom="90dp"
                  android:background="@drawable/circle_red"
                  android:stateListAnimator="@null"
                  android:text="✆"
                  android:textColor="#FFFFFF"
                  android:textSize="30sp"
                  android:visibility="gone" />
          </LinearLayout>
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/res/drawable"
          cat > "raksha/app/src/main/res/drawable/btn_bg.xml" <<'RAKSHA_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <shape xmlns:android="http://schemas.android.com/apk/res/android" android:shape="rectangle">
              <solid android:color="#24242E" />
              <corners android:radius="12dp" />
              <stroke android:width="1dp" android:color="#2A2A36" />
          </shape>
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/res/drawable"
          cat > "raksha/app/src/main/res/drawable/btn_green.xml" <<'RAKSHA_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <shape xmlns:android="http://schemas.android.com/apk/res/android" android:shape="rectangle">
              <solid android:color="#1E9E64" />
              <corners android:radius="12dp" />
          </shape>
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/res/drawable"
          cat > "raksha/app/src/main/res/drawable/btn_red.xml" <<'RAKSHA_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <shape xmlns:android="http://schemas.android.com/apk/res/android" android:shape="rectangle">
              <solid android:color="#E5293F" />
              <corners android:radius="12dp" />
          </shape>
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/res/drawable"
          cat > "raksha/app/src/main/res/drawable/btn_pink.xml" <<'RAKSHA_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <shape xmlns:android="http://schemas.android.com/apk/res/android" android:shape="rectangle">
              <solid android:color="#FF6B8B" />
              <corners android:radius="12dp" />
          </shape>
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/res/drawable"
          cat > "raksha/app/src/main/res/drawable/field_bg.xml" <<'RAKSHA_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <shape xmlns:android="http://schemas.android.com/apk/res/android" android:shape="rectangle">
              <solid android:color="#16161E" />
              <corners android:radius="10dp" />
              <stroke android:width="1dp" android:color="#2A2A36" />
          </shape>
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/res/drawable"
          cat > "raksha/app/src/main/res/drawable/sos_bg.xml" <<'RAKSHA_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <shape xmlns:android="http://schemas.android.com/apk/res/android" android:shape="oval">
              <solid android:color="#E5293F" />
              <stroke android:width="14dp" android:color="#66E5293F" />
          </shape>
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/res/drawable"
          cat > "raksha/app/src/main/res/drawable/circle_red.xml" <<'RAKSHA_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <shape xmlns:android="http://schemas.android.com/apk/res/android" android:shape="oval">
              <solid android:color="#E5293F" />
          </shape>
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/res/drawable"
          cat > "raksha/app/src/main/res/drawable/circle_green.xml" <<'RAKSHA_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <shape xmlns:android="http://schemas.android.com/apk/res/android" android:shape="oval">
              <solid android:color="#22B573" />
          </shape>
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/res/drawable"
          cat > "raksha/app/src/main/res/drawable/circle_avatar.xml" <<'RAKSHA_EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <shape xmlns:android="http://schemas.android.com/apk/res/android" android:shape="oval">
              <solid android:color="#3A4A5C" />
          </shape>
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/res/drawable"
          cat > "raksha/app/src/main/res/drawable/ic_shield.xml" <<'RAKSHA_EOF'
          <vector xmlns:android="http://schemas.android.com/apk/res/android"
              android:width="24dp"
              android:height="24dp"
              android:viewportWidth="24"
              android:viewportHeight="24">
              <path
                  android:fillColor="#FFFFFFFF"
                  android:pathData="M12,1L3,5v6c0,5.55 3.84,10.74 9,12 5.16,-1.26 9,-6.45 9,-12V5L12,1z" />
          </vector>
          RAKSHA_EOF
          mkdir -p "raksha/app/src/main/res/drawable"
          cat > "raksha/app/src/main/res/drawable/ic_launcher.xml" <<'RAKSHA_EOF'
          <vector xmlns:android="http://schemas.android.com/apk/res/android"
              android:width="108dp"
              android:height="108dp"
              android:viewportWidth="24"
              android:viewportHeight="24">
              <path
                  android:fillColor="#FFE5293F"
                  android:pathData="M12,0a12,12 0,1 1,0 24a12,12 0,1 1,0 -24z" />
              <group
                  android:pivotX="12"
                  android:pivotY="12"
                  android:scaleX="0.62"
                  android:scaleY="0.62">
                  <path
                      android:fillColor="#FFFFFFFF"
                      android:pathData="M12,1L3,5v6c0,5.55 3.84,10.74 9,12 5.16,-1.26 9,-6.45 9,-12V5L12,1z" />
                  <path
                      android:fillColor="#FFE5293F"
                      android:pathData="M12,6.5c-1.1,0 -2,0.9 -2,2s0.9,2 2,2 2,-0.9 2,-2 -0.9,-2 -2,-2zM12,11.5c-2,0 -4,1 -4,2.5v1h8v-1c0,-1.5 -2,-2.5 -4,-2.5z" />
              </group>
          </vector>
          RAKSHA_EOF

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '8.7'

      - name: APK build karo
        working-directory: raksha
        run: gradle assembleDebug --no-daemon --stacktrace

      - name: APK ka naam
        run: cp raksha/app/build/outputs/apk/debug/app-debug.apk ANAS-RAKSHA.apk

      - name: APK upload (Actions)
        uses: actions/upload-artifact@v4
        with:
          name: ANAS-RAKSHA-APK
          path: ANAS-RAKSHA.apk

      - name: APK Release mein daalo (direct download link)
        uses: softprops/action-gh-release@v2
        with:
          tag_name: v1.0.${{ github.run_number }}
          name: ANAS RAKSHA v1.0.${{ github.run_number }}
          body: "ANAS RAKSHA - Women Safety App. Made by Anas. ANAS-RAKSHA.apk download karke install karo."
          files: ANAS-RAKSHA.apk
