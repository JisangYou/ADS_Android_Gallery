## 수업 내용

- 권한 설정 후에 ▽
- 카메라 앱 띄우고 사진 불러오는 학습
- 안드로이드 내장 앨범에서 사진 불러오는 학습

## Code Review

### BaseActivity

```Java
public abstract class BaseActivity extends AppCompatActivity {

    private static final int REQ_CODE = 999;
    private static final String permissions[] = {
            Manifest.permission.WRITE_EXTERNAL_STORAGE,
            Manifest.permission.CAMERA
    };

    public abstract void init();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 앱 버전 체크 - 호환성 처리
        if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.M){
            checkPermission();
        }else{
            init();
        }
    }

    @TargetApi(Build.VERSION_CODES.M)
    private void checkPermission(){
        // 권한에 대한 승인 여부
        List<String> requires = new ArrayList<>();
        for(String permission : permissions){
            if(checkSelfPermission(permission) != PackageManager.PERMISSION_GRANTED){
                requires.add(permission);
            }
        }
        // 승인이 안된 권한이 있을 경우만 승인 요청
        if(requires.size() > 0){
            String perms[] = requires.toArray(new String[requires.size()]);
            requestPermissions(perms, REQ_CODE);
        }else {
            init();
        }
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        // 권한 승인 여부 체크
        if(requestCode == REQ_CODE){
            boolean granted = true;
            for(int grant : grantResults){
                if(grant != PackageManager.PERMISSION_GRANTED){
                    granted = false;
                    break;
                }
            }
            if(granted){
                init();
            }
        }
    }
}

```
### MainActivity

```Java
public class MainActivity extends BaseActivity {

    ImageView imageView;

    @Override
    public void init() {
        setContentView(R.layout.activity_main);
        imageView = (ImageView) findViewById(R.id.imageView);
    }

    // 저장된 파일의 경로를 가지는 컨텐츠 Uri
    Uri fileUri = null;
    public void onCamera(View view){
        // 카메라 앱 띄우서 결과 이미지 저장하기
        // 1. Intent 만들기
        Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
        // 2. 호환성 처리 버전체크 - 롤리팝 이상
        if(Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP){
            try {
                // 3. 실제 파일이 저장되는 파일 객체 < 빈 파일을 생성해 둔다
                // 3.1. 실제 파일이 저장되는 곳에 권한이 부여되어 있어야 한다.
                //      롤리팝 부터는 File Provider를 선언해 줘야만한다. > Manifest에
                File photoFile = createFile();
                fileUri = FileProvider.getUriForFile(this, BuildConfig.APPLICATION_ID+".provider", photoFile);
                intent.putExtra(MediaStore.EXTRA_OUTPUT, fileUri);
                startActivityForResult(intent, REQ_CAMERA);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }else{
            startActivityForResult(intent, REQ_CAMERA);
        }
    }

    // 이미지를 저장하기 위해 쓰기 권한이 있는 빈 파일을 생성해두는 함수
    private File createFile() throws IOException{
        // 임시파일명 생성
        String tempFileName = "Temp_"+System.currentTimeMillis();
        // 임시파일 저장용 디렉토리 생성
        File tempDir = new File(Environment.getExternalStorageDirectory() + "/CameraN/");
        // 생성체크
        if(!tempDir.exists()){
            tempDir.mkdirs();
        }
        // 실제 임시파일을 생성
        File tempFile = File.createTempFile(
                tempFileName,  // 파일명
                ".jpg",  // 확장자
                tempDir   // 저장될 디렉토리
        );
        return tempFile;
    }



    private static final int REQ_CAMERA = 222;
    private static final int REQ_GALLERY = 333;

    public void onGallery(View view){
        // 인텐트를 사용해서 갤러리 액티비티 호출
        Intent intent = new Intent(Intent.ACTION_PICK, MediaStore.Images.Media.EXTERNAL_CONTENT_URI);
        startActivityForResult(intent, REQ_GALLERY);
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if(resultCode == RESULT_OK) {
            Uri imageUri = null;
            switch (requestCode) {
                // 갤러리 액티비티 종료시 호출
                case REQ_GALLERY:
                    if(data != null) {
                        // data에 값이 있으면 갤러리에서 선택한 이미지를
                        // 이미지뷰에 세팅해준다.
                        imageUri = data.getData();
                        imageView.setImageURI(imageUri);
                    }
                    break;
                case REQ_CAMERA:
                    // 롤리팝 미만 버전 체크
                    if(Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP){
                        imageUri = data.getData();
                    } else {
                        imageUri = fileUri;
                    }
                    imageView.setImageURI(imageUri);
                    break;
            }
        }
    }
}
```



## 보충설명

### Intent 

- 인텐트는 자신이 만든 앱안에서 활동하는 것 뿐만 아니라 내가 만들지 않은 타 애플리케이션의 기능을 수행

> 즉, 안드로이드 시스템은 내가 만든 인텐트의 정보를 처리하면서 내가 만든 액티비키나 애플리케이션의 구성요소가 해야할 일을 지정하는 것 이외에도 타 애플리케이션의 기능을 수행하는 등 훨씬 유연한 기능의 애플리케이션을 만들 수 있게 함.

- 인텐트의 기본 구성요소로 __액션(Action)__과 __데이터(Data)__가 존재
- 액션은 수행할 기능이며, 데이터는 액션이 수행될 대상 데이터를 의미
```Java
Intent intent = new Intent(Intent.ACTION_DIAL, Uri.parse(data)); 

// 액션은 ACTION_DIAL 전화다이얼을 걸어라라는 액션이며,
// data값을 uri로 파싱한 Uri.parse(data)는 것은 액션이 수행할 data 즉, 전화번호
// Uri로 파싱한 전화번호 data를 대상으로 전화다이얼을 걸어라. 라는 뜻! 
// 이 뜻을 인텐트에 담아 안드로이드 시스템에게 전달
```

#### 1. 명시적 Intent 2. 암시적 Intent

> 명시적 인텐트는 인텐트에 클래스 객체나 컴포넌트 이름을 지정하여 호출할 대상을 확실히 알 수 있는 경우에 사용
- 주로 애플리케이션 내부에서 사용합니다. 
- 명시적 인텐르를 사용하는 이유론 특정 컴포넌트나 액티비티가 명확하게 실행되어야할 경우에 사용

> 인텐트의 액션과 데이터를 지정하긴 했지만, 호출할 대상이 달라질 수 있는 경우에는 암시적 인텐트를 사용합니다.

- 설치된 애플리케이션들에 대한 정보를 알고 있는 안드로이드 시스템이 인텐트를 이용해 요청한 정보를 처리할 수 있는 적절한 콤포넌트를 찾아본 다음 사용자에게 그 대상과 처리 결과를 보여주는 과정을 거치게 됨.
- 이미 기존에 어떤 기능들을 지원하는 앱들이 있는 경우에 암시적 인텐트를 사용해서 그 앱들을 사용하면 되는 것

### 출처

- 출처 : http://limkydev.tistory.com/35
- 참고 : Do it! 안드로이드 앱 프로그래밍 서적
- 참고 : http://bitsoul.tistory.com/132
- 참고 : http://arabiannight.tistory.com


## TODO

- 다음과 같은 로직들의 의미 및 확장된 기능 들 학습하기.
```Java 
FileProvider.getUriForFile(this, BuildConfig.APPLICATION_ID+".provider", photoFile);
```
```Java
File tempFile = File.createTempFile(
                tempFileName,  // 파일명
                ".jpg",  // 확장자
                tempDir   // 저장될 디렉토리
        );
```
## Retrospect

- 카메라앱을 사용해서 이미지를 저장할 때, File을 다루거나, crop을 다루거나, resizing을 다루는 등 커스터마이징이 필요할 수 있을 것이므로
이 점 TODO에서 작성 한것처럼 해당 기능에 대해 깊고 넓게 학습할 것.

## Output
- 생략