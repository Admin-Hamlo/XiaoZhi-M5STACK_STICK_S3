# XiaoZhi-M5STACK_STICK_S3
Тюнинг исходного проекта под конкретный стик. Вайбкодинг, чисто для себя.
Русский интерфейс, горизонтальный дисплей, исправлена громкость и текст в 3 строки.
Бинарник готов к прошивке.


Сводка изменений для M5Stack StickS3 (xiaozhi-esp32)


1. Конфигурация сборки (sdkconfig / menuconfig)

• Целевой чип и плата: esp32s3, CONFIG_BOARD_TYPE_M5STACK_STICK_S3=y.
• Флеш-память и разметка: размер флеша 8 MB (CONFIG_ESPTOOLPY_FLASHSIZE_8MB=y), таблица разделов 8m.csv.
• Активация голосом (Wake Word): включен AFE Wake Word (CONFIG_USE_AFE_WAKE_WORD / WAKE_WORD_TYPE), выбрана полноразмерная модель wn9_nihaoxiaozhi_tts («Нихао Сяочжи»).
• Интерфейс: язык Russian (ru-RU), стиль сообщений Default message style (CONFIG_USE_WECHAT_MESSAGE_STYLE выключен), включен многострочный вывод текста CONFIG_USE_MULTILINE_CHAT_MESSAGE=y.
• Сетевые буферы (стабильный звук при высоком пинге): увеличены CONFIG_LWIP_UDP_RECVMBOX_SIZE=32, CONFIG_ESP_WIFI_RX_BA_WIN=16, CONFIG_ESP_WIFI_STATIC_RX_BUFFER_NUM=16, CONFIG_ESP_WIFI_DYNAMIC_RX_BUFFER_NUM=32.

2. Звук: буферы, громкость и отключение микрофона при речи

• main/audio/audio_service.h (строки 39–42): увеличены очереди аудиопакетов для защиты от сетевых просадок:cpp
#define MAX_ENCODE_TASKS_IN_QUEUE 8
#define MAX_PLAYBACK_TASKS_IN_QUEUE 10
#define MAX_DECODE_PACKETS_IN_QUEUE (4800 / OPUS_FRAME_DURATION_MS)
#define MAX_SEND_PACKETS_IN_QUEUE (6000 / OPUS_FRAME_DURATION_MS)
Use code with caution.
• main/audio/audio_codec.cc (функция AudioCodec::Start): стартовая громкость при включении ограничена безопасным значением 50% (вместо старых 255 из NVS), с принудительным вызовом SetOutputVolume(output_volume_) при старте.
• main/boards/m5stack/stick-s3/m5stack_stick_s3.cc:
	• Подключен <cmath>.
	• В классе Sticks3AudioCodec::SetOutputVolume заменена линейная формула (* 0.6f) на логарифмическую/степенную кривую в диапазоне от 30 (минимум при volume > 0) до 75 (безопасный максимум без перегруза усилителя и перезагрузок по питанию):cpp
int hw_volume = 0;
if (volume > 0) {
    float normalized = volume / 100.0f;
    hw_volume = (int)(30.0f + 45.0f * std::pow(normalized, 1.8f) + 0.5f);
}
Es8311AudioCodec::SetOutputVolume(hw_volume);
output_volume_ = hw_volume;
Use code with caution.
	• В InitializePm1() включен стабильный буст питания: pmic_->setBoostEnable(true);.
• main/application.cc (функция HandleStateChangedEvent, ветка kDeviceStateSpeaking): отключено фоновое прослушивание AFE/WakeNet во время речи ассистента (audio_service_.EnableWakeWordDetection(false);), чтобы исключить акустическую обратную связь и нагрузку на шину/CPU.
• main/audio/codecs/es8311_audio_codec.cc (функция CreateDuplexChannels): увеличены DMA-буферы шины I2S до .dma_desc_num = 16 и .dma_frame_num = 480.

3. Экран: альбомный поворот на 90°, шрифты и 3 строки текста

• main/boards/m5stack/stick-s3/config.h: дисплей переведен в горизонтальный режим 240×135 с пересчитанными смещениями матрицы ST7789:cpp
#define DISPLAY_WIDTH           240
#define DISPLAY_HEIGHT          135
#define DISPLAY_MIRROR_X        true
#define DISPLAY_MIRROR_Y        false
#define DISPLAY_SWAP_XY         true
#define DISPLAY_OFFSET_X        40
#define DISPLAY_OFFSET_Y        53
Use code with caution.
• main/CMakeLists.txt (секция CONFIG_BOARD_TYPE_M5STACK_STICK_S3): основной шрифт диалога увеличен до font_noto_sans_basic_20_4 (20 px с 4-битным сглаживанием).
• main/display/lcd_display.cc:
	• Объявлен LV_FONT_DECLARE(font_noto_sans_basic_16_4); и применен к верхней строке состояния (status_label_ и notification_label_), чтобы она оставалась компактной.
	• В ветке #else (SetupUI() → #if CONFIG_USE_MULTILINE_CHAT_MESSAGE) нижней плашке bottom_bar_ и метке chat_message_label_ задана фиксированная высота под 3 строки крупного текста (text_font->line_height * 3 + 8), режим LV_LABEL_LONG_WRAP, межстрочный интервал 0 и выравнивание текста по верхнему краю (LV_ALIGN_TOP_MID).
