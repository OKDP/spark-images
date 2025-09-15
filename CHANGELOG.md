# Changelog

## [2.0.0](https://github.com/OKDP/spark-images/compare/v1.0.1...v2.0.0) (2025-09-15)


### Miscellaneous Chores

* release 2.0.0 ([e177b9f](https://github.com/OKDP/spark-images/commit/e177b9f7787060013d7fb7043c69a91ba4972da9))

## [1.0.1](https://github.com/OKDP/spark-images/compare/v1.0.0...v1.0.1) (2024-06-10)


### Bug Fixes

* **okdp-addons/okdp-spark-auth-filter:** Improve management of IDP supported scopes and offline_access OKDP/okdp-spark-auth-filter[#16](https://github.com/OKDP/spark-images/issues/16) ([2310f1c](https://github.com/OKDP/spark-images/commit/2310f1c36b2a02c1db4b61d75b4b3bf206fa944e))


### Miscellaneous Chores

* release 1.0.1 ([5436219](https://github.com/OKDP/spark-images/commit/54362192d0b1146d6a01b40fece338bba3f341c0))

## 1.0.0 (2024-04-04)


### Features

* **spark-base:** Add new spark-base image (java/scala only) without okdp extensions ([59eb7d8](https://github.com/OKDP/spark-images/commit/59eb7d8530a79efb0b84396256625c087fc9e25e))
* **spark-py:** Add new spark-py image with python basic requirements ([ad94e07](https://github.com/OKDP/spark-images/commit/ad94e07da550eb00e2228b539d166450ef2e1a4d))
* **spark-r:** Add new spark-r image with R basic requirements ([a9c0880](https://github.com/OKDP/spark-images/commit/a9c0880ebdd7c67c09e4150740bfb0be57a9d9f3))
* **spark:** Add new spark image (java/scala only) with okdp extensions (aws/minio, prometheus java agent, okdp-spark-auth-filter) ([1d935f7](https://github.com/OKDP/spark-images/commit/1d935f744ede3160eba6fb4d05dd8b71e9bed991))
* **spark:** Minimize minio/aws sdk v1/v2 depedendencies to reduce spark image size ([5da5b8a](https://github.com/OKDP/spark-images/commit/5da5b8a734c40d178e35d7981434afc66fc29604))


### Bug Fixes

* **spark-base:** Add missing gpg keys in the spark project release keys ([980b001](https://github.com/OKDP/spark-images/commit/980b0011181f23ed6867b1e4c32489f8904b3543))
