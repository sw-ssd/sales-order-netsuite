# 商店上線前檢查清單

## Android (Google Play)

- [ ] keystore 備份確認（`android/keystore/hexagon-salesorder-keystore.jks`）
- [ ] `key.properties` 內容正確
- [ ] Prod flavor 指向正式 API endpoint
- [ ] Firebase Crashlytics 在 prod 正常收到 crash report
- [ ] Firebase Analytics 確認不記錄 PII
- [ ] 深層連結（customer QR code signin）release build 測試通過
- [ ] Play Store 隱私政策 URL 設定完成
- [ ] Data Safety 表單填寫完成
- [ ] 內容分級問卷完成
- [ ] 商店圖文（截圖、feature graphic、描述）上傳完成
- [ ] Closed track 內部測試通過（10+ 測試者）
- [ ] Open track beta 測試通過

## iOS (App Store)

- [ ] `PrivacyInfo.xcprivacy` 內容正確
- [ ] App Store Connect 隱私標籤填寫完成
- [ ] App Store 描述（繁中/英文）設定完成
- [ ] App Store 截圖上傳完成（含 6.9" artboard 成品）
- [ ] TestFlight 內部測試通過
- [ ] TestFlight 外部測試通過
- [ ] App Store 審查提交

## 通用

- [ ] 隱私政策網頁可公開存取
- [ ] 聯絡信箱可正常收信
- [ ] 後端 API prod endpoint 穩定運作
- [ ] 版本號正確（pubspec.yaml 及 git tag）
