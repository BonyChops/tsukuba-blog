---
title: "FastAPIで諸々セットアップ"
date: 2023-05-17T13:32:30+09:00
draft: false
tags: ["enPiT", "enPiT進捗報告"]
---

こちらは 2023/05/17(水)の enPiT 進捗報告になります．  
今日は~~システム演習で使う~~FastAPI で

![](/~s2313881/blog/images/2023-05-19-build-fastapi/system.png)

## Auth Tester

Firebase Authentication で得た idtoken を用いてデバッグするにあたって，idtoken を気軽に発行したい．  
idtoken は基本フロントエンドで発行することを想定されているので，Node.js で発行しようとすると少し面倒．  
なら，そのままフロントエンドで作るのが最適と考えた．

- 当初は u.tsukuba.ac.jp の MS 垢(みんなもってるから)にしようと思ってた
  - まじか...
    ![](/~s2313881/blog/images/2023-05-19-build-fastapi/ms-fail.png)
    - 使えないっぽい
- ということで，klis の Google アカウントを使うことにした

`App.jsx`

```jsx
import { GoogleAuthProvider, OAuthProvider } from "firebase/auth";
import { signInWithPopup } from "firebase/auth";
import { auth } from "./firebase";
import { useEffect, useState } from "react";
function App() {
  const [user, setUser] = useState(null);
  useEffect(() => {
    // startAuth();
    // When State Changed
    auth.onAuthStateChanged((user) => {
      if (user) {
        setUser(user);
        console.log(user);
      } else {
        console.log("Not logged in");
      }
    });
  }, []);

  const startAuth = () => {
    // Provider: Google
    const provider = new GoogleAuthProvider(auth);

    // Popup
    signInWithPopup(auth, provider);
  };

  return (
    <div className="App">
      <div style={{ textAlign: "center", marginTop: "10px" }}>
        <h1>Auth Tester</h1>
        {user ? (
          <div>
            <h2>{user.reloadUserInfo.displayName}としてログイン済み</h2>
            <button
              onClick={() => {
                auth.signOut();
                setUser(null);
              }}
            >
              ログアウト
            </button>
            <p>
              <h3>基本情報</h3>
              <table style={{ marginLeft: "auto", marginRight: "auto" }}>
                <tbody>
                  <tr>
                    <th>uid</th>
                    <td>{user.uid}</td>
                  </tr>
                  <tr>
                    <th>email</th>
                    <td>{user.email}</td>
                  </tr>
                </tbody>
              </table>
            </p>
            <h3>Token</h3>
            <div
              style={{
                width: "500px",
                marginLeft: "auto",
                marginRight: "auto",
                overflowWrap: "break-word",
              }}
            >
              {user.accessToken}
            </div>
          </div>
        ) : (
          <button
            onClick={() => {
              startAuth();
            }}
          >
            Googleアカウントでログイン
          </button>
        )}
      </div>
    </div>
  );
}

export default App;
```

![](/~s2313881/blog/images/2023-05-19-build-fastapi/auth-tester.png)

- あとは Actions でデプロイ

  - デプロイ先は u.tsukuba.ac.jp みたいに VPS なので rsync で
  - `DocumentRoot`変えておく  
    `/etc/apache2/site-available/000-*.conf`

    ```diff
    <IfModule mod_ssl.c>
    <VirtualHost *:443>
            # ...
    -       DocumentRoot /var/www/html
    +       DocumentRoot /home/bonychops/public_html_global
            # ...
            </Directory>
    </VirtualHost>
    </IfModule>
    ```

  - `.github/workflows/deploy.yml`
    - `Firebase Config`忘れてた(下部は修正済み)

  ```yml
    name: Deploy

    on:
    push:
        branches:
        - main # Set a branch name to trigger deployment

    jobs:
    deploy:
        runs-on: ubuntu-22.04
        steps:
        - uses: actions/checkout@v3
            with:
            submodules: true # Fetch Hugo themes (true OR recursive)
            fetch-depth: 0 # Fetch all history for .GitInfo and .Lastmod
        - uses: actions/setup-node@v3
            with:
            node-version: 20
        - name: Set Firebase Config
            run: |
            echo '$FIREBASE_CONFIG' | envsubst > src/firebaseConfig.json
            env:
            FIREBASE_CONFIG: ${{ secrets.FIREBASE_CONFIG }}
        - name: Build
            run: npm ci && npm run build

        - name: Prepare SSH Secret
            run: |
            mkdir -p ~/.ssh
            chmod 700 ~/.ssh
            echo '$SSH_SECRET' | envsubst > ~/.ssh/id_ed25519
            echo '$SSH_CONFIG' | envsubst > ~/.ssh/config
            chmod 600 ~/.ssh/id_ed25519
            ssh-keyscan -t rsa github.com >> ~/.ssh/known_hosts
            env:
            SSH_SECRET: ${{ secrets.SSH_SECRET }}
            SSH_CONFIG: ${{ secrets.SSH_CONFIG }}

        - name: Deploy
            run: |
            rsync -e "ssh -o 'StrictHostKeyChecking no'" -avz --delete build/ klis-vps:$SSH_PATH
            env:
            SSH_PATH: ${{ secrets.SSH_PATH }}

  ```

  - `Require all granted`忘れてた  
    `/etc/apache2/site-available/000-*.conf`

    ```diff
    + <Directory "/home/bonychops/public_html_pub">
    +     Require all granted
    + </Directory>
    ```

  - `package.json`に`homepage足す`

    ```diff
      {
          "name": "impact-z2-auth-tester",
          "version": "0.1.0",
    +     "homepage": "https://example.com/auth-tester/",
          ...
      }

    ```

    - Firebase に認証済みドメインとして登録...
    - 思ったより時間がかかったができた
