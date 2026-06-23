# 📷 EXIF Viewer & Remover - 기술 문서

이 문서는 이미지 메타데이터 뷰어 및 정리기의 기술적 구현 세부사항을 설명합니다. EXIF 파일 형식의 구조, XMP/IPTC/ICC 확장 메타데이터 파싱, Canvas API를 통한 메타데이터 정리 원리 등 핵심 기술들을 다룹니다.

---

## 📚 목차

1. [EXIF 파일 형식](#1-exif-파일-형식)
2. [EXIF 데이터 구조](#2-exif-데이터-구조)
3. [주요 EXIF 태그](#3-주요-exif-태그)
4. [GPS 데이터 처리](#4-gps-데이터-처리)
5. [exifr 라이브러리 활용](#5-exifr-라이브러리-활용)
6. [XMP / IPTC / ICC 처리](#6-xmp--iptc--icc-처리)
7. [Canvas를 통한 메타데이터 제거](#7-canvas를-통한-메타데이터-제거)
8. [파일 형식별 처리](#8-파일-형식별-처리)
9. [프라이버시 고려사항](#9-프라이버시-고려사항)
10. [성능 최적화](#10-성능-최적화)
11. [한계점 및 개선 방향](#11-한계점-및-개선-방향)

---

## 1. EXIF 파일 형식

### 1.1 EXIF란?

**EXIF (Exchangeable Image File Format)**는 디지털 카메라와 스캐너에서 생성되는 이미지 파일에 메타데이터를 저장하기 위한 표준입니다.

```
┌─────────────────────────────────────────────────────────────┐
│                    JPEG 파일 구조                            │
├─────────────────────────────────────────────────────────────┤
│ SOI (Start of Image)  │ FF D8                               │
├───────────────────────┼─────────────────────────────────────┤
│ APP1 (EXIF Data)      │ FF E1 [length] [EXIF data]          │
├───────────────────────┼─────────────────────────────────────┤
│ APP0 (JFIF)           │ FF E0 [JFIF data] (optional)        │
├───────────────────────┼─────────────────────────────────────┤
│ DQT, DHT, SOF, SOS    │ 이미지 데이터                        │
├───────────────────────┼─────────────────────────────────────┤
│ EOI (End of Image)    │ FF D9                               │
└───────────────────────┴─────────────────────────────────────┘
```

### 1.2 EXIF의 역사

| 버전 | 연도 | 주요 변경 |
|------|------|----------|
| 1.0 | 1995 | 최초 발표 (JEIDA) |
| 2.0 | 1997 | 썸네일 압축 지원 |
| 2.1 | 1998 | FlashPix 확장 |
| 2.2 | 2002 | 컬러 스페이스 확장 |
| 2.21 | 2003 | Adobe RGB 지원 |
| 2.3 | 2010 | GPS 정밀도 향상 |
| 2.32 | 2019 | UTF-8 지원 |

### 1.3 EXIF가 포함되는 파일 형식

```
┌──────────────┬─────────────────────────────────────────┐
│ 파일 형식     │ EXIF 지원 방식                          │
├──────────────┼─────────────────────────────────────────┤
│ JPEG         │ APP1 세그먼트에 직접 포함                │
│ TIFF         │ IFD (Image File Directory) 구조          │
│ PNG          │ eXIf 청크 (PNG 1.5+)                     │
│ WebP         │ EXIF 청크 (VP8X 확장)                    │
│ HEIF/HEIC    │ meta 박스 내 Exif 아이템                 │
└──────────────┴─────────────────────────────────────────┘
```

---

## 2. EXIF 데이터 구조

### 2.1 TIFF 헤더

EXIF 데이터는 TIFF (Tagged Image File Format) 구조를 기반으로 합니다.

```
┌────────────────────────────────────────────────────────────┐
│                      TIFF 헤더 (8 bytes)                   │
├──────────────┬─────────┬───────────────────────────────────┤
│ Byte Order   │ 2 bytes │ 0x4949 (Little Endian, "II")      │
│              │         │ 0x4D4D (Big Endian, "MM")         │
├──────────────┼─────────┼───────────────────────────────────┤
│ TIFF Magic   │ 2 bytes │ 0x002A (42)                       │
├──────────────┼─────────┼───────────────────────────────────┤
│ IFD Offset   │ 4 bytes │ 첫 번째 IFD까지의 오프셋          │
└──────────────┴─────────┴───────────────────────────────────┘
```

### 2.2 IFD (Image File Directory)

```
┌────────────────────────────────────────────────────────────┐
│                    IFD 구조                                │
├──────────────┬─────────┬───────────────────────────────────┤
│ Entry Count  │ 2 bytes │ IFD 엔트리 개수                   │
├──────────────┼─────────┼───────────────────────────────────┤
│ IFD Entry 1  │ 12 bytes│ 태그 정보                         │
│ IFD Entry 2  │ 12 bytes│                                   │
│ ...          │         │                                   │
├──────────────┼─────────┼───────────────────────────────────┤
│ Next IFD     │ 4 bytes │ 다음 IFD 오프셋 (0이면 끝)        │
└──────────────┴─────────┴───────────────────────────────────┘
```

### 2.3 IFD Entry 구조

```
┌────────────────────────────────────────────────────────────┐
│                 IFD Entry (12 bytes)                       │
├──────────────┬─────────┬───────────────────────────────────┤
│ Tag ID       │ 2 bytes │ 태그 식별자 (예: 0x010F = Make)   │
├──────────────┼─────────┼───────────────────────────────────┤
│ Data Type    │ 2 bytes │ 데이터 타입                       │
│              │         │ 1=BYTE, 2=ASCII, 3=SHORT,         │
│              │         │ 4=LONG, 5=RATIONAL, ...           │
├──────────────┼─────────┼───────────────────────────────────┤
│ Count        │ 4 bytes │ 값의 개수                         │
├──────────────┼─────────┼───────────────────────────────────┤
│ Value/Offset │ 4 bytes │ 값 또는 값이 저장된 오프셋        │
└──────────────┴─────────┴───────────────────────────────────┘
```

### 2.4 데이터 타입

```javascript
const EXIF_DATA_TYPES = {
    1: { name: 'BYTE', size: 1 },
    2: { name: 'ASCII', size: 1 },
    3: { name: 'SHORT', size: 2 },
    4: { name: 'LONG', size: 4 },
    5: { name: 'RATIONAL', size: 8 },     // 2개의 LONG (분자/분모)
    6: { name: 'SBYTE', size: 1 },
    7: { name: 'UNDEFINED', size: 1 },
    8: { name: 'SSHORT', size: 2 },
    9: { name: 'SLONG', size: 4 },
    10: { name: 'SRATIONAL', size: 8 },   // 2개의 SLONG
    11: { name: 'FLOAT', size: 4 },
    12: { name: 'DOUBLE', size: 8 }
};
```

### 2.5 IFD 체인

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   IFD0      │───→│   IFD1      │───→│   NULL      │
│ (Main)      │    │ (Thumbnail) │    │             │
└─────┬───────┘    └─────────────┘    └─────────────┘
      │
      ├───→ EXIF IFD (촬영 정보)
      │
      └───→ GPS IFD (위치 정보)
```

---

## 3. 주요 EXIF 태그

### 3.1 카메라 정보 태그 (IFD0)

| 태그 ID | 태그명 | 설명 | 예시 값 |
|---------|--------|------|---------|
| 0x010F | Make | 카메라 제조사 | "Apple" |
| 0x0110 | Model | 카메라 모델 | "iPhone 14 Pro" |
| 0x0131 | Software | 소프트웨어 | "iOS 17.1" |
| 0x0132 | DateTime | 수정 일시 | "2024:01:15 14:30:00" |
| 0x013B | Artist | 작가 | "John Doe" |
| 0x8298 | Copyright | 저작권 | "© 2024" |

### 3.2 촬영 정보 태그 (EXIF IFD)

```javascript
const EXIF_TAGS = {
    0x829A: 'ExposureTime',      // 노출 시간 (예: 1/60)
    0x829D: 'FNumber',           // 조리개 (예: f/2.8)
    0x8822: 'ExposureProgram',   // 노출 프로그램
    0x8827: 'ISOSpeedRatings',   // ISO 감도
    0x9003: 'DateTimeOriginal',  // 촬영 일시
    0x9004: 'DateTimeDigitized', // 디지털화 일시
    0x9201: 'ShutterSpeedValue', // APEX 셔터 속도
    0x9202: 'ApertureValue',     // APEX 조리개 값
    0x9203: 'BrightnessValue',   // 밝기 값
    0x9204: 'ExposureBiasValue', // 노출 보정
    0x9205: 'MaxApertureValue',  // 최대 조리개
    0x9207: 'MeteringMode',      // 측광 모드
    0x9208: 'LightSource',       // 광원
    0x9209: 'Flash',             // 플래시
    0x920A: 'FocalLength',       // 초점 거리
    0xA001: 'ColorSpace',        // 색상 공간
    0xA002: 'ExifImageWidth',    // 이미지 너비
    0xA003: 'ExifImageHeight',   // 이미지 높이
    0xA402: 'ExposureMode',      // 노출 모드
    0xA403: 'WhiteBalance',      // 화이트 밸런스
    0xA405: 'FocalLengthIn35mm', // 35mm 환산 초점거리
    0xA406: 'SceneCaptureType',  // 장면 유형
};
```

### 3.3 GPS 태그

```javascript
const GPS_TAGS = {
    0x0000: 'GPSVersionID',      // GPS 버전
    0x0001: 'GPSLatitudeRef',    // 위도 참조 (N/S)
    0x0002: 'GPSLatitude',       // 위도 [도, 분, 초]
    0x0003: 'GPSLongitudeRef',   // 경도 참조 (E/W)
    0x0004: 'GPSLongitude',      // 경도 [도, 분, 초]
    0x0005: 'GPSAltitudeRef',    // 고도 참조 (해수면 위/아래)
    0x0006: 'GPSAltitude',       // 고도
    0x0007: 'GPSTimeStamp',      // GPS 시간 (UTC)
    0x0008: 'GPSSatellites',     // 사용된 위성 정보
    0x0009: 'GPSStatus',         // GPS 상태
    0x000A: 'GPSMeasureMode',    // 측정 모드 (2D/3D)
    0x000B: 'GPSDOP',            // 정밀도 (DOP)
    0x000C: 'GPSSpeedRef',       // 속도 단위
    0x000D: 'GPSSpeed',          // 속도
    0x000E: 'GPSTrackRef',       // 방향 참조
    0x000F: 'GPSTrack',          // 이동 방향
    0x0010: 'GPSImgDirectionRef',// 이미지 방향 참조
    0x0011: 'GPSImgDirection',   // 이미지 방향
    0x001D: 'GPSDateStamp',      // GPS 날짜
};
```

---

## 4. GPS 데이터 처리

### 4.1 DMS (도분초) 형식

GPS 좌표는 EXIF에서 **DMS (Degrees, Minutes, Seconds)** 형식으로 저장됩니다.

```
예: 서울시청 좌표
위도: 37° 33' 59.5" N
경도: 126° 58' 40.4" E

EXIF 저장 형식:
GPSLatitude: [37, 33, 59.5]      // 세 개의 RATIONAL 값
GPSLatitudeRef: "N"
GPSLongitude: [126, 58, 40.4]
GPSLongitudeRef: "E"
```

### 4.2 DMS → DD 변환

**DD (Decimal Degrees)**는 지도 서비스에서 사용하는 십진수 형식입니다.

```javascript
/**
 * DMS(도분초)를 DD(십진도)로 변환
 * 
 * 공식: DD = D + M/60 + S/3600
 * 
 * @param {number[]} dms - [도, 분, 초] 배열
 * @param {string} ref - 방향 참조 ('N', 'S', 'E', 'W')
 * @returns {number} - 십진도
 */
function convertDMSToDD(dms, ref) {
    if (!dms || dms.length < 3) return 0;
    
    // 도 + 분/60 + 초/3600
    let dd = dms[0] + (dms[1] / 60) + (dms[2] / 3600);
    
    // 남위(S) 또는 서경(W)이면 음수
    if (ref === 'S' || ref === 'W') {
        dd = -dd;
    }
    
    return dd;
}

// 예시: 서울시청
// DMS: 37° 33' 59.5" N
// DD: 37 + 33/60 + 59.5/3600 = 37.5665°
```

### 4.3 DD → DMS 역변환

```javascript
/**
 * DD(십진도)를 DMS(도분초)로 변환
 */
function convertDDToDMS(dd) {
    const d = Math.floor(Math.abs(dd));
    const mFloat = (Math.abs(dd) - d) * 60;
    const m = Math.floor(mFloat);
    const s = (mFloat - m) * 60;
    
    return { degrees: d, minutes: m, seconds: s };
}
```

### 4.4 Google Maps 링크 생성

```javascript
function createGoogleMapsLink(lat, lng) {
    // 기본 링크
    const basicLink = `https://www.google.com/maps?q=${lat},${lng}`;
    
    // 줌 레벨 포함 링크
    const detailedLink = `https://www.google.com/maps/@${lat},${lng},15z`;
    
    // 스트리트 뷰 링크
    const streetViewLink = `https://www.google.com/maps/@?api=1&map_action=pano&viewpoint=${lat},${lng}`;
    
    return basicLink;
}
```

### 4.5 GPS 정밀도

```
GPS 좌표 자릿수와 정밀도:
┌──────────────┬──────────────────────┬──────────────┐
│ 소수점 자릿수 │ 정밀도               │ 예시         │
├──────────────┼──────────────────────┼──────────────┤
│ 0            │ ±111 km              │ 37°         │
│ 1            │ ±11.1 km             │ 37.5°       │
│ 2            │ ±1.1 km              │ 37.56°      │
│ 3            │ ±111 m               │ 37.566°     │
│ 4            │ ±11 m                │ 37.5665°    │
│ 5            │ ±1.1 m               │ 37.56654°   │
│ 6            │ ±0.11 m (11 cm)      │ 37.566540°  │
└──────────────┴──────────────────────┴──────────────┘

스마트폰 GPS 일반 정밀도: 소수점 5-6자리 (1-10m 오차)
```

---

## 5. exifr 라이브러리 활용

### 5.1 라이브러리 로드

```html
<!-- 로컬 번들 로드 -->
<script src="./vendor/exifr.full.umd.js"></script>
```

### 5.2 기본 사용법

```javascript
/**
 * 이미지에서 메타데이터 추출
 */
async function extractExifData(file) {
    return await exifr.parse(file, true);
}

// 사용 예시
const file = document.getElementById('myFileInput').files[0];
const exifData = await extractExifData(file);
```

### 5.3 특정 태그 읽기

```javascript
const data = await exifr.parse(file, true);

// 개별 태그 읽기
const make = data.Make;
const model = data.Model;
const dateTime = data.DateTimeOriginal;

// GPS 좌표 읽기
const gpsLat = data.GPSLatitude;
const gpsLatRef = data.GPSLatitudeRef;
const gpsLng = data.GPSLongitude;
const gpsLngRef = data.GPSLongitudeRef;
```

### 5.4 EXIF 데이터 분류

```javascript
/**
 * EXIF 데이터를 카테고리별로 분류
 */
function categorizeExifData(exifData) {
    return {
        camera: {
            make: exifData.Make,
            model: exifData.Model,
            software: exifData.Software,
            lens: exifData.LensModel
        },
        dateTime: {
            modified: exifData.DateTime,
            original: exifData.DateTimeOriginal,
            digitized: exifData.DateTimeDigitized
        },
        settings: {
            exposureTime: exifData.ExposureTime,
            fNumber: exifData.FNumber,
            iso: exifData.ISOSpeedRatings,
            focalLength: exifData.FocalLength,
            flash: exifData.Flash,
            whiteBalance: exifData.WhiteBalance
        },
        gps: {
            latitude: exifData.GPSLatitude,
            latitudeRef: exifData.GPSLatitudeRef,
            longitude: exifData.GPSLongitude,
            longitudeRef: exifData.GPSLongitudeRef,
            altitude: exifData.GPSAltitude
        }
    };
}
```

### 5.5 플래시 상태 해석

```javascript
/**
 * 플래시 값을 사람이 읽을 수 있는 형태로 변환
 * 
 * 플래시 비트 구조:
 * bit 0: 플래시 발광 여부
 * bit 1-2: 리턴 상태
 * bit 3-4: 플래시 모드
 * bit 5: 플래시 기능 존재
 * bit 6: 적목 감소 모드
 */
function interpretFlash(flashValue) {
    const flashModes = {
        0x00: '플래시 없음',
        0x01: '발광',
        0x05: '발광, 리턴 없음',
        0x07: '발광, 리턴 감지',
        0x08: '발광 안함 (강제)',
        0x09: '발광 (강제)',
        0x0D: '발광 (강제), 리턴 없음',
        0x0F: '발광 (강제), 리턴 감지',
        0x10: '발광 안함',
        0x18: '자동, 발광 안함',
        0x19: '자동, 발광',
        0x1D: '자동, 발광, 리턴 없음',
        0x1F: '자동, 발광, 리턴 감지',
        0x20: '플래시 기능 없음',
        0x41: '발광, 적목 감소',
        0x45: '발광, 적목 감소, 리턴 없음',
        0x47: '발광, 적목 감소, 리턴 감지',
        0x49: '발광 (강제), 적목 감소',
        0x4D: '발광 (강제), 적목 감소, 리턴 없음',
        0x4F: '발광 (강제), 적목 감소, 리턴 감지',
        0x59: '자동, 발광, 적목 감소',
        0x5D: '자동, 발광, 적목 감소, 리턴 없음',
        0x5F: '자동, 발광, 적목 감소, 리턴 감지'
    };
    
    return flashModes[flashValue] || `알 수 없음 (0x${flashValue.toString(16)})`;
}
```

---

## 6. XMP / IPTC / ICC 처리

`exifr.parse(file, true)`는 EXIF 외에도 XMP, IPTC, ICC, JFIF, PNG IHDR 같은 확장 메타데이터를 함께 반환할 수 있습니다. 이 프로젝트는 그 결과를 카테고리별로 분리해 보여줍니다.

### 6.1 XMP

XMP는 편집 도구와 워크플로 정보를 담는 메타데이터입니다. Lightroom, Photoshop, 카메라 앱, 에디터가 남긴 흔적이 여기에 들어갈 수 있습니다.

대표적으로 다음 값을 보여줍니다.

- `CreatorTool`
- `Label`
- `Rating`
- `Description`
- `Title`
- `Subject`
- `Rights`

### 6.2 IPTC

IPTC는 사진 설명, 캡션, 키워드, 저자 정보를 담는 데 자주 쓰입니다.

대표적으로 다음 값을 보여줍니다.

- `headline`
- `caption`
- `credit`
- `byline`
- `keywords`
- `category`
- `captionWriter`

### 6.3 ICC

ICC 프로파일은 색상 재현을 위한 정보입니다. 개인정보는 아니지만, 색상 정확도와 편집 워크플로 판단에 유용합니다.

대표적으로 다음 값을 보여줍니다.

- `ProfileDescription`
- `ProfileCMMType`
- `ProfileClass`
- `ColorSpaceData`
- `RenderingIntent`
- `DeviceModel`
- `DeviceManufacturer`

---

## 6. Canvas를 통한 메타데이터 제거

### 6.1 원리

Canvas API를 통해 이미지를 다시 그리면 **원본 파일의 대부분의 메타데이터가 제거**됩니다. Canvas는 순수한 픽셀 데이터만 처리하기 때문입니다.

```
원본 이미지              Canvas 처리              결과 이미지
┌─────────────────┐      ┌─────────────┐      ┌─────────────────┐
│ EXIF 메타데이터  │      │ drawImage() │      │                 │
│ - GPS 정보      │  ──→ │             │ ──→  │ 픽셀 데이터만    │
│ - 카메라 정보    │      │ toBlob()    │      │ (메타데이터 없음) │
│ - 날짜 정보     │      │             │      │                 │
│ 픽셀 데이터     │      └─────────────┘      └─────────────────┘
└─────────────────┘
```

### 6.2 구현 코드

```javascript
/**
 * Canvas를 통해 이미지에서 EXIF 메타데이터 제거
 * 
 * @param {HTMLImageElement} image - 원본 이미지 엘리먼트
 * @param {File} originalFile - 원본 파일 (형식 판단용)
 * @returns {Promise<Blob>} - 메타데이터가 제거된 이미지 Blob
 */
async function removeExifMetadata(image, originalFile) {
    return new Promise((resolve, reject) => {
        const canvas = document.createElement('canvas');
        const ctx = canvas.getContext('2d');
        
        // 캔버스 크기를 이미지 크기로 설정
        canvas.width = image.naturalWidth;
        canvas.height = image.naturalHeight;
        
        // 이미지 그리기 (이 과정에서 EXIF 제거됨)
        ctx.drawImage(image, 0, 0);
        
        // 출력 형식 및 품질 결정
        const isJPEG = originalFile.type === 'image/jpeg' || 
                       originalFile.type === 'image/jpg';
        const mimeType = isJPEG ? 'image/jpeg' : 'image/png';
        const quality = isJPEG ? 0.92 : undefined;  // PNG는 무손실
        
        // Blob으로 변환
        canvas.toBlob((blob) => {
            if (blob) {
                resolve(blob);
            } else {
                reject(new Error('이미지 변환 실패'));
            }
        }, mimeType, quality);
    });
}
```

### 6.3 JPEG 품질 설정

```javascript
/**
 * JPEG 품질과 파일 크기의 관계
 * 
 * 품질 1.0  : 원본과 거의 동일, 파일 크기 최대
 * 품질 0.92 : 시각적으로 거의 동일, 파일 크기 ~70-80%
 * 품질 0.85 : 약간의 압축 아티팩트, 파일 크기 ~50-60%
 * 품질 0.75 : 눈에 띄는 압축 아티팩트, 파일 크기 ~40%
 */

// 권장 품질 설정
const JPEG_QUALITY = {
    high: 0.92,      // 고품질 (메타데이터 제거 목적)
    medium: 0.85,    // 중간 품질 (웹 사용)
    low: 0.75        // 낮은 품질 (썸네일)
};
```

### 6.4 처리 흐름

```javascript
async function processImage() {
    // 1. 파일 읽기
    const file = input.files[0];
    const dataURL = await readFileAsDataURL(file);
    
    // 2. 이미지 로드
    const img = await loadImage(dataURL);
    
    // 3. EXIF 추출 (표시용)
    const exifData = await extractExifData(img);
    displayExifData(exifData);
    
    // 4. 메타데이터 제거
    const cleanBlob = await removeExifMetadata(img, file);
    
    // 5. 크기 비교
    console.log(`원본: ${file.size} bytes`);
    console.log(`정리 후: ${cleanBlob.size} bytes`);
    console.log(`감소: ${((1 - cleanBlob.size / file.size) * 100).toFixed(1)}%`);
    
    // 6. 다운로드
    downloadBlob(cleanBlob, `${file.name}_clean`);
}
```

### 6.5 Blob 다운로드

```javascript
/**
 * Blob을 파일로 다운로드
 */
function downloadBlob(blob, filename) {
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = filename;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);  // 메모리 해제
}
```

---

## 7. 파일 형식별 처리

### 7.1 JPEG

```
JPEG 특성:
- EXIF 데이터가 APP1 세그먼트에 저장
- 손실 압축 → 재압축 시 품질 손실 발생
- Canvas 출력 시 적절한 품질 설정 필요

권장 처리:
- 품질 0.92로 재인코딩 (시각적 손실 최소화)
- 파일 크기는 원본보다 약간 작아질 수 있음
```

### 7.2 PNG

```
PNG 특성:
- EXIF는 eXIf 청크에 저장 (PNG 1.5+)
- 무손실 압축 → 재압축해도 품질 손실 없음
- 알파 채널(투명도) 지원

권장 처리:
- 무손실로 재인코딩
- 알파 채널 보존
```

### 7.3 WebP

```javascript
// WebP 처리
async function processWebP(image) {
    const canvas = document.createElement('canvas');
    canvas.width = image.width;
    canvas.height = image.height;
    
    const ctx = canvas.getContext('2d');
    ctx.drawImage(image, 0, 0);
    
    // WebP로 출력 (지원되는 경우)
    if (canvas.toDataURL('image/webp').startsWith('data:image/webp')) {
        return await canvasToBlob(canvas, 'image/webp', 0.92);
    } else {
        // WebP 미지원 브라우저에서는 PNG로 폴백
        return await canvasToBlob(canvas, 'image/png');
    }
}
```

### 7.4 형식 감지

```javascript
/**
 * 파일의 실제 형식 감지 (MIME type 외에 매직 바이트 확인)
 */
async function detectImageFormat(file) {
    const buffer = await file.slice(0, 12).arrayBuffer();
    const bytes = new Uint8Array(buffer);
    
    // JPEG: FF D8 FF
    if (bytes[0] === 0xFF && bytes[1] === 0xD8 && bytes[2] === 0xFF) {
        return 'image/jpeg';
    }
    
    // PNG: 89 50 4E 47 0D 0A 1A 0A
    if (bytes[0] === 0x89 && bytes[1] === 0x50 && bytes[2] === 0x4E && 
        bytes[3] === 0x47) {
        return 'image/png';
    }
    
    // WebP: 52 49 46 46 ... 57 45 42 50
    if (bytes[0] === 0x52 && bytes[1] === 0x49 && bytes[2] === 0x46 && 
        bytes[3] === 0x46 && bytes[8] === 0x57 && bytes[9] === 0x45) {
        return 'image/webp';
    }
    
    return file.type;  // 폴백
}
```

---

## 8. 프라이버시 고려사항

### 8.1 민감한 EXIF 태그

```javascript
const SENSITIVE_TAGS = [
    // GPS 정보 (위치 노출)
    'GPSLatitude', 'GPSLongitude', 'GPSLatitudeRef', 'GPSLongitudeRef',
    'GPSAltitude', 'GPSAltitudeRef', 'GPSTimeStamp', 'GPSDateStamp',
    
    // 시간 정보 (일상 패턴 추론 가능)
    'DateTime', 'DateTimeOriginal', 'DateTimeDigitized',
    
    // 기기 정보 (개인 식별 가능)
    'Make', 'Model', 'Software', 'LensModel',
    
    // 시리얼 번호 (기기 추적 가능)
    'BodySerialNumber', 'LensSerialNumber', 'CameraSerialNumber',
    
    // 소유자 정보
    'Artist', 'Copyright', 'OwnerName', 'CameraOwnerName',
    
    // 고유 식별자
    'ImageUniqueID', 'SerialNumber'
];
```

### 8.2 위험도 분류

```
┌────────────────┬───────────────────────────────────────────┐
│ 위험도         │ 태그                                       │
├────────────────┼───────────────────────────────────────────┤
│ 🔴 높음        │ GPS 좌표, 주소, 정확한 위치 정보           │
│                │ → 집/직장 위치 노출, 스토킹 위험            │
├────────────────┼───────────────────────────────────────────┤
│ 🟠 중간        │ 촬영 날짜/시간, 시리얼 번호                │
│                │ → 일상 패턴 추론, 기기 추적 가능            │
├────────────────┼───────────────────────────────────────────┤
│ 🟡 낮음        │ 카메라 모델, ISO, 조리개 등                │
│                │ → 간접적 식별 가능성                        │
└────────────────┴───────────────────────────────────────────┘
```

### 8.3 100% 클라이언트 사이드 처리

```javascript
/*
 * 보안 모델:
 * 
 * 1. 이미지 데이터는 서버로 업로드하지 않음
 *    - 이미지가 서버로 업로드되지 않음
 *    - XHR/Fetch 요청 없음
 * 
 * 2. 로컬 저장 없음
 *    - localStorage/sessionStorage 미사용
 *    - IndexedDB 미사용
 *    - 파일 시스템 쓰기 없음 (다운로드 제외)
 * 
 * 3. 메모리 내 처리
 *    - FileReader API: 파일 → 메모리
 *    - Canvas API: 이미지 처리
 *    - Blob API: 결과 생성
 *    - URL.createObjectURL: 임시 URL 생성
 *    - URL.revokeObjectURL: 메모리 해제
 */
```

---

## 9. 성능 최적화

### 9.1 대용량 이미지 처리

```javascript
/**
 * 대용량 이미지의 경우 리사이즈 옵션 제공
 */
function processLargeImage(image, maxDimension = 4096) {
    const { width, height } = image;
    
    if (width <= maxDimension && height <= maxDimension) {
        return { width, height, scaled: false };
    }
    
    const scale = Math.min(maxDimension / width, maxDimension / height);
    
    return {
        width: Math.round(width * scale),
        height: Math.round(height * scale),
        scaled: true,
        originalWidth: width,
        originalHeight: height
    };
}
```

### 9.2 메모리 관리

```javascript
/**
 * 메모리 누수 방지를 위한 정리
 */
function cleanup() {
    // Object URL 해제
    if (currentObjectURL) {
        URL.revokeObjectURL(currentObjectURL);
        currentObjectURL = null;
    }
    
    // Canvas 메모리 해제
    const canvas = document.getElementById('canvas');
    canvas.width = 0;
    canvas.height = 0;
    
    // 이미지 참조 해제
    currentImage = null;
    currentFile = null;
}
```

### 9.3 비동기 처리

```javascript
/**
 * 큰 이미지 처리 시 UI 블로킹 방지
 */
async function processImageAsync(file) {
    // UI 업데이트 허용
    await new Promise(resolve => setTimeout(resolve, 0));
    
    // 진행률 표시
    updateProgress(10, '파일 읽는 중...');
    
    const dataURL = await readFileAsDataURL(file);
    updateProgress(30, '이미지 로딩 중...');
    
    const image = await loadImage(dataURL);
    updateProgress(50, 'EXIF 분석 중...');
    
    const exifData = await extractExifData(image);
    updateProgress(70, '메타데이터 제거 중...');
    
    const cleanBlob = await removeExifMetadata(image, file);
    updateProgress(100, '완료!');
    
    return { exifData, cleanBlob };
}
```

---

## 10. 한계점 및 개선 방향

### 10.1 현재 구현의 한계

| 한계 | 설명 | 개선 방안 |
|------|------|----------|
| **JPEG 재압축** | 재압축 시 미세한 품질 손실 | 더 높은 품질 설정 옵션 |
| **특수 형식** | HEIC, RAW 미지원 | 라이브러리 추가 |
| **XMP 데이터** | 일부 XMP 메타데이터 미지원 | XMP 파서 추가 |
| **벌크 처리** | 대량 파일 처리 미지원 | 배치 처리 기능 추가 |

### 10.2 고급 기능 제안

#### 1) 선택적 메타데이터 제거

```javascript
// 특정 태그만 유지하고 나머지 제거
const keepTags = ['ExposureTime', 'FNumber', 'ISO'];

// 특정 태그만 제거
const removeTags = ['GPSLatitude', 'GPSLongitude', 'DateTime'];
```

#### 2) 벌크 처리

```javascript
async function processBatch(files) {
    const results = [];
    
    for (let i = 0; i < files.length; i++) {
        updateProgress(i / files.length * 100);
        const result = await processImage(files[i]);
        results.push(result);
    }
    
    // ZIP으로 묶어서 다운로드
    return createZipFromBlobs(results);
}
```

#### 3) EXIF 편집 (메타데이터 수정)

```javascript
// 향후 구현 가능: 특정 필드 수정 후 저장
// 주의: 이미지 이진 데이터 직접 조작 필요
```

### 10.3 로드맵

```
┌─────────────────────────────────────────────────────────┐
│                     로드맵                              │
├─────────────────────────────────────────────────────────┤
│ v1.1: 벌크 처리 (다중 파일 일괄 처리)                   │
│ v1.2: 선택적 메타데이터 제거                            │
│ v1.3: HEIC/HEIF 지원                                   │
│ v2.0: 메타데이터 편집 기능                              │
│ v2.1: 클라우드 저장소 연동 (선택적)                     │
└─────────────────────────────────────────────────────────┘
```

---

## 부록: EXIF 태그 전체 목록

### A. IFD0 태그 (이미지 정보)

| 태그 ID | 태그명 | 타입 |
|---------|--------|------|
| 0x0100 | ImageWidth | LONG |
| 0x0101 | ImageHeight | LONG |
| 0x0102 | BitsPerSample | SHORT |
| 0x0103 | Compression | SHORT |
| 0x010E | ImageDescription | ASCII |
| 0x010F | Make | ASCII |
| 0x0110 | Model | ASCII |
| 0x0112 | Orientation | SHORT |
| 0x011A | XResolution | RATIONAL |
| 0x011B | YResolution | RATIONAL |
| 0x0128 | ResolutionUnit | SHORT |
| 0x0131 | Software | ASCII |
| 0x0132 | DateTime | ASCII |

### B. EXIF IFD 태그 (촬영 정보)

| 태그 ID | 태그명 | 타입 |
|---------|--------|------|
| 0x829A | ExposureTime | RATIONAL |
| 0x829D | FNumber | RATIONAL |
| 0x8822 | ExposureProgram | SHORT |
| 0x8827 | ISOSpeedRatings | SHORT |
| 0x9000 | ExifVersion | UNDEFINED |
| 0x9003 | DateTimeOriginal | ASCII |
| 0x9004 | DateTimeDigitized | ASCII |
| 0x9201 | ShutterSpeedValue | SRATIONAL |
| 0x9202 | ApertureValue | RATIONAL |
| 0x9203 | BrightnessValue | SRATIONAL |
| 0x9204 | ExposureBiasValue | SRATIONAL |
| 0x9205 | MaxApertureValue | RATIONAL |
| 0x9207 | MeteringMode | SHORT |
| 0x9209 | Flash | SHORT |
| 0x920A | FocalLength | RATIONAL |

---

## 참고 자료

1. **EXIF 2.32 Specification** - CIPA DC-008-Translation-2019
2. **TIFF 6.0 Specification** - Adobe
3. **exifr GitHub** - https://github.com/MikeKovarik/exifr
4. **Canvas API - MDN** - https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API
5. **JPEG File Format** - ITU-T T.81

---

*이 문서는 EXIF Viewer & Remover 프로젝트의 기술적 구현 세부사항을 설명합니다.*
