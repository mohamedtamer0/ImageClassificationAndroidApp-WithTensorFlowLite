# ImageClassificationAndroidApp-WithTensorFlowLite


## Preview
<div align="center">
<br/>
  


Camera            |   Gallery
:-------------------------:|:-------------------------:
<img src="https://user-images.githubusercontent.com/51374446/152708687-12016b48-3375-4067-a747-1a5369932b19.gif" width="200" height="400" />  | <img src="https://user-images.githubusercontent.com/51374446/152708817-85af648f-a116-40da-9d42-8006bc0e941a.gif" width="200" height="400" /> 

 </div>
  
```groovy
implementation(Deps.coreKts)
implementation(Deps.appCompat)
implementation(Deps.material)
implementation(Deps.constraintLayout)
testImplementation(Deps.jUnit)
androidTestImplementation(Deps.jUnitExt)
androidTestImplementation(Deps.espresso)
implementation (Deps.tensorflowSupport)
implementation (Deps.tensorflowMetaData)

```

```kotlin
  private lateinit var imageView: ImageView
  private lateinit var button: Button
  private lateinit var tvOutput: TextView
  private val GALLERY_REQUEST_CODE = 123
```


```kotlin
  imageView = binding.imageView
  button = binding.btnCaptureImage
  tvOutput = binding.tvOutput
  val buttonLoad = binding.btnLoadImage

  button.setOnClickListener {
      if (ContextCompat.checkSelfPermission(
              this,
              Manifest.permission.CAMERA
          ) == PackageManager.PERMISSION_GRANTED
      ) {
          takePicturePreview.launch(null)
      } else {
          requestPermission.launch(Manifest.permission.CAMERA)
      }
  }

  buttonLoad.setOnClickListener {
      if (ContextCompat.checkSelfPermission(
              this,
              Manifest.permission.READ_EXTERNAL_STORAGE
          ) == PackageManager.PERMISSION_GRANTED
      ) {
          val intent =
              Intent(Intent.ACTION_PICK, MediaStore.Images.Media.EXTERNAL_CONTENT_URI)
          intent.type = "image/*"
          val mimeTypes = arrayOf("image/jpeg", "image/png", "image/jpg")
          intent.putExtra(Intent.EXTRA_MIME_TYPES, mimeTypes)
          intent.flags = Intent.FLAG_GRANT_READ_URI_PERMISSION
          onresult.launch(intent)
      } else {
          requestPermission.launch(Manifest.permission.READ_EXTERNAL_STORAGE)
      }
  }


  //to redirect user to google search for the scientific name
  tvOutput.setOnClickListener {
      val intent = Intent(
          Intent.ACTION_VIEW,
          Uri.parse("https://www.google.com/search?q=${tvOutput.text}")
      )
      startActivity(intent)
  }


  // to download image when longPress on ImageView
  imageView.setOnLongClickListener {
      requestPermissionLauncher.launch(Manifest.permission.WRITE_EXTERNAL_STORAGE)
      return@setOnLongClickListener true
  }

```

```kotlin
    //request camera permission
    private val requestPermission =
        registerForActivityResult(ActivityResultContracts.RequestPermission()) { granted ->
            if (granted) {
                takePicturePreview.launch(null)
            } else {
                Toast.makeText(this, "Permission Denied !! Try again", Toast.LENGTH_SHORT).show()
            }
        }
```

```kotlin
    //launch camera and take picture
    private val takePicturePreview =
        registerForActivityResult(ActivityResultContracts.TakePicturePreview()) { bitmap ->
            if (bitmap != null) {
                imageView.setImageBitmap(bitmap)
                outputGenerator(bitmap)
            }
        }
```

```kotlin
    //to get image from gallery
    private val onresult =
        registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { result ->
            Log.i("TAG", "This is the result: ${result.data} ${result.resultCode}")
            onResultReceived(GALLERY_REQUEST_CODE, result)
        }
```

```kotlin
    private fun onResultReceived(requestCode: Int, result: ActivityResult?) {
        when (requestCode) {
            GALLERY_REQUEST_CODE -> {
                if (result?.resultCode == Activity.RESULT_OK) {
                    result.data?.data?.let { uri ->
                        Log.i("TAG", "onResultReceived: $uri")
                        val bitmap =
                            BitmapFactory.decodeStream(contentResolver.openInputStream(uri))
                        imageView.setImageBitmap(bitmap)
                        outputGenerator(bitmap)
                    }
                } else {
                    Log.e("TAG", "onActivityResult: error in selecting image")
                }
            }
        }
    }
```

```kotlin
    private fun outputGenerator(bitmap: Bitmap) {
//declaring tensor flow lite model variable
        val birdModel = BridsModel.newInstance(this)

        // converting bitmap into tensor flow image
        val newBitmap = bitmap.copy(Bitmap.Config.ARGB_8888, true)
        val tfimage = TensorImage.fromBitmap(newBitmap)

        //process the image using trained model and sort it in descending order
        val outputs1 = birdModel.process(tfimage)
            .probabilityAsCategoryList.apply {
                sortByDescending { it.score }
            }
        //getting result having high Probability
        val highProbabilityOutput1 = outputs1[0]

        //setting output text
        tvOutput.text = highProbabilityOutput1.label
        Log.i(
            "TAG", "outputGenerator: $highProbabilityOutput1"
        )
    }
```


```kotlin
    // fun that takes a bitmap and store to user's device
    private fun downloadImage(mBitmap: Bitmap): Uri? {
        val contentValues = ContentValues().apply {
            put(
                MediaStore.Images.Media.DISPLAY_NAME,
                "Birds_Images" + System.currentTimeMillis() / 1000
            )
            put(MediaStore.Images.Media.MIME_TYPE, "image/png")
        }
        val uri = MediaStore.Images.Media.EXTERNAL_CONTENT_URI
        if (uri != null) {
            contentResolver.insert(uri, contentValues)?.also {
                contentResolver.openOutputStream(it).use { outputStream ->
                    if (!mBitmap.compress(Bitmap.CompressFormat.PNG, 100, outputStream)) {
                        throw IOException("Couldn't save the bitmap")
                    } else {
                        Toast.makeText(
                            applicationContext,
                            "Image Saved",
                            Toast.LENGTH_LONG
                        ).show()
                    }
                }
                return it
            }
        }
        return null
    }
```
