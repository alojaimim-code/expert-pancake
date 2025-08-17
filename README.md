# expert-pancake
# تثبيت Flutter (غالبًا القالب عنده مسبقًا، لكن نضمن)
git clone https://github.com/flutter/flutter.git -b stable
echo 'export PATH="$PATH:/workspaces/mkhmkh-local/flutter/bin"' >> ~/.bashrc
source ~/.bashrc
flutter --version

# إنشاء مشروع جديد باسم game_local
flutter create game_local
cd game_local

# تشغيله كتطبيق ويب (يطلع لك رابط للمعاينة)
flutter run -d web-server --web-hostname 0.0.0.0 --web-port 8080