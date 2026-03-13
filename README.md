original 3DS是現行3DS2.2架構的proverif。假設商家允許使用者使用FIDO登入，並將這個登入成功的結果傳給銀行端進行風險評估。結果證明使用者有正確執行登入，商家有驗證以及銀行也有收到驗證結果，但是商家可能轉送假的驗證結果給銀行。

<img width="904" height="143" alt="image" src="https://github.com/user-attachments/assets/b8ab05fc-4066-483a-adba-358ef7c9efb5" />


3DSFIDO_1是改進架構的proverif，包含FIDO註冊、EMV卡片綁定及3DS交易。在 Dolev–Yao 模型下，本文協定至少形式化滿足以下性質：registration authenticity、binding authenticity、transaction authentication、injective agreement / replay resistance，以及關鍵長期秘密與金鑰的機密性。

<img width="1437" height="486" alt="image" src="https://github.com/user-attachments/assets/58571e2e-461f-4945-9ac3-d35a09c7baf0" />



myFIDO是我們文中使用到的FIDO註冊及驗證200次的時間統計。
