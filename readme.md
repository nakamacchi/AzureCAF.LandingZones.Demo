# Azure CAF Landing Zones 設計・構築ハンズオン

## 作業デモビデオ

| ビデオ | ビデオ時間 | 作業時間目安 | リンク |
| :-- | :-- | :-- | :-- |
|ESLZDemo_00_作業の事前準備|27:57|1 時間|[リンク](https://livesend.microsoft.com/i/KiIa1FQzy1DUXI8U0n7t8Mk08Fb9jKY3D9OXIRgzmtzxN49yn37eJrnD5f1FLIqDji0dQfZDXWsAPEYrpn9qH4S0xuFNa9lIXx9Lt2NEJfn0bo2GrOG48sGCYb1QcjEM)|
|ESLZDemo_01_Step00_環境準備|37:10|1 時間|[リンク](https://livesend.microsoft.com/i/KiIa1FQzy1DUXI8U0n7t8Mk08Fb9jKY3D9OXIRgzmtzWlOQl0ryRu6r1Sn6PLUSSIGNeTdZOx4PklHFnyMPhwzQymXm7tEf___T___aQE3zJTD7tFWQmwRM8BCh6xAXFUu3Hi2IGuVa)|
|ESLZDemo_02_Step01_初期環境セットアップ|37:08|1 時間|[リンク](https://livesend.microsoft.com/i/KiIa1FQzy1DUXI8U0n7t8Mk08Fb9jKY3D9OXIRgzmty3lIBTiHyk___aKePLUSSIGNrkA2HFWaGbK7G9Z9RK1vDfGJnim249___gyoafziCsydTeqFwGLOeyW0Nt1cMo5iYJBG4UuLK)|
|ESLZDemo_03_Step02_管理サブスクリプションの作成（前半）|41:05|2 時間|[リンク](https://livesend.microsoft.com/i/KiIa1FQzy1DUXI8U0n7t8Mk08Fb9jKY3D9OXIRgzmtyuIJHJGxHlXyruUKSZFgoiqoFQtKM6WGa10g0wqigG8cHQR___uaePLUSSIGNR9z7PAyAeXXSzesby___8TNKxHk62WCS2GjO)|
|ESLZDemo_04_Step02_管理サブスクリプションの作成（後半）|10:03|1 時間|[リンク](https://livesend.microsoft.com/i/KiIa1FQzy1DUXI8U0n7t8Mk08Fb9jKY3D9OXIRgzmtzNFsLwDgF7pYpQiEpJcyCweH5A5cPLUSSIGNY2uXXOZkdcKBgnPl4OV0EZTIrP5KYzb7aiH5m2cxJODFpfd9P4lB6WiTt)|
|ESLZDemo_05_Step03_ハブサブスクリプションの作成|03:18|1 時間|[リンク](https://livesend.microsoft.com/i/KiIa1FQzy1DUXI8U0n7t8Mk08Fb9jKY3D9OXIRgzmtzxN49yn37eJrnD5f1FLIqDbPLUSSIGNPvjzSMxMSUEUBuXMUsYoGrCnsnKD5oxJ5TUqaACKvon6ewZ4tGMhTc6t6tj8j0)|
|ESLZDemo_06_Step04_管理基盤の構成設定|28:40|1 時間|[リンク](https://livesend.microsoft.com/i/KiIa1FQzy1DUXI8U0n7t8Mk08Fb9jKY3D9OXIRgzmtwb9ehaqUR1bVgNjwxUn8lZ67jW3rgXfFbLq218H___G2aegx9BiqJlzdJz2YIMNsvbSJjaXZydgEvBKD6lJRlRi8)|
|ESLZDemo_07_Step05_仮想マシンのコンプライアンス対応|44:26|2 時間|[リンク](https://livesend.microsoft.com/i/KiIa1FQzy1DUXI8U0n7t8Mk08Fb9jKY3D9OXIRgzmtyuIJHJGxHlXyruUKSZFgoiXIXOZV1xgqlaxE8xRWGe___UO3627judlDM3UnmqX12p7IlXq0GbS47AhbjnOK___jsL)|
|ESLZDemo_08_Step06_SpokeA_IaaS型WebDBインフラの作成|25:22|2 時間|[リンク](https://livesend.microsoft.com/i/KiIa1FQzy1DUXI8U0n7t8Mk08Fb9jKY3D9OXIRgzmtxc0hP9Z1kKTECqbtEgiraQlsD1sM1Ok0zH9nZkpDSN___x7gKeOpxawUKFn532drkexZmq2zohKkQVenXIdrIcC___)|
|ESLZDemo_09_Step07_SpokeA_イントラWebアプリのセットアップ|47:08|3 時間|[リンク](https://livesend.microsoft.com/i/KiIa1FQzy1DUXI8U0n7t8Mk08Fb9jKY3D9OXIRgzmtxQXPLUSSIGN___dalE8F4n68O7GPED3c0FWx0wdeHVe2V7AzflG___EFx76Vn7___V8NiGKXEuCH8lt1EOWCZBD5WPLUSSIGN378pUcFHe)|
|ESLZDemo_10_Step08_SpokeB_PaaS型WebDBインフラの作成|30:11|1 時間|[リンク](https://livesend.microsoft.com/i/KiIa1FQzy1DUXI8U0n7t8Mk08Fb9jKY3D9OXIRgzmtw6G7iQYzLWpOMJ73X83AsFq5y8eAuxNsapqnLIkTPNWf0qCrPLUSSIGN3d5dlZg5czcrTqZ3___vhhYf9e6r___nU5ZTrtrcv)|
|ESLZDemo_11_Step09・10_SpokeB_WebアプリのセットアップとDMZWAFの作成|24:47|1 時間|[リンク](https://livesend.microsoft.com/i/KiIa1FQzy1DUXI8U0n7t8Mk08Fb9jKY3D9OXIRgzmtwb9ehaqUR1bVgNjwxUn8lZggMKPLUSSIGNk7fN8B___kaWnKpiiBjRvGIuiH0xJ63d9Jmu2g7NvcAqNmRmtTayPyQZNC3UI)|
|ESLZDemo_12_Step11_AzurePolicyガバナンスとMDfCセキュリティ(前半)|25:31|1 時間|[リンク](https://livesend.microsoft.com/i/KiIa1FQzy1DUXI8U0n7t8Mk08Fb9jKY3D9OXIRgzmtwb9ehaqUR1bVgNjwxUn8lZJY93mIkPLUSSIGNHatQRQSxegbvacyTFt1o8tnNqk5MRXKd___SbqHC66JpMmg4M5cPLUSSIGNre3ucR)|
|ESLZDemo_13_Step12_AzurePolicyガバナンスとMDfCセキュリティ(後半)|39:31|2 時間|[リンク](https://livesend.microsoft.com/i/KiIa1FQzy1DUXI8U0n7t8Mk08Fb9jKY3D9OXIRgzmtw6G7iQYzLWpOMJ73X83AsFwVBCosV6r7yhXDeKFSz26X7XNHEV2FLazNVId7pYkv4oAtFUvtglvyv78ZaClXl___)|
|ESLZDemo_14_Step13_運用監視（モニタリング）|34:41|1 時間|[リンク](https://livesend.microsoft.com/i/KiIa1FQzy1DUXI8U0n7t8Mk08Fb9jKY3D9OXIRgzmtzxN49yn37eJrnD5f1FLIqDqmTK8rH9sN7GlJnK7vj6v06iDkoPOcMtwlIk2i6kSIlz6jAKieqAdRhALIhmiohR)|
|ESLZDemo_15_後片付け方法とまとめ|11:56|1 時間|[リンク](https://livesend.microsoft.com/i/KiIa1FQzy1DUXI8U0n7t8Mk08Fb9jKY3D9OXIRgzmtw6G7iQYzLWpOMJ73X83AsFj2hHHnDLAoWax6t5ZPCHwlEWXsunU4xC9gcbFz2HLomcsJbbW5CWZc0ncZWPRTGW)|

## 関連資料

| 資料名 | リンク |
| :-- | :-- |
| デモビデオ解説資料 | [リンク](https://livesend.microsoft.com/i/KiIa1FQzy1DUXI8U0n7t8Mk08Fb9jKY3D9OXIRgzmtw6G7iQYzLWpOMJ73X83AsFsEFNlzuWWX33ZeMbHAmCICLEiCQGOKRUGseM___zBc376Orq7Ohi8WkCCPLUSSIGN8IzIIJFy) |
| 共通基盤設計ガイド（作成途中バージョン） | [リンク](https://livesend.microsoft.com/i/KiIa1FQzy1DUXI8U0n7t8Mk08Fb9jKY3D9OXIRgzmtw6G7iQYzLWpOMJ73X83AsFmmPLUSSIGNB7a1iYsI6AZxvTP0___Qq41JtPLUSSIGNSmSMJkBmjV3S5X7euwMAyDyYri1yLiqUJPLUSSIGNr4T) |
| 共通基盤設計 ガバナンス設計 Excel シート | [リンク](https://livesend.microsoft.com/i/KiIa1FQzy1DUXI8U0n7t8Mk08Fb9jKY3D9OXIRgzmtxQXPLUSSIGN___dalE8F4n68O7GPED3PLUSSIGNEd1DnAGagmPOLSkuUD7Qpm5ms0xyKUHsAMqcJquJoBtxjPLUSSIGNs5kN15NOhEcTyIgWW) |
