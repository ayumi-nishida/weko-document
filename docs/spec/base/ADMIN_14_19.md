### Shibboleth

- > 目的・用途

  本画面の機能は以下の通りである。<br>

  ・[システム利用者がログインする際のシボレスユーザーの許可／拒否を設定](#シボレスユーザーの許可拒否を設定)<br>
  ・[システム利用者のデフォルトロールの設定](#デフォルトロールの設定)<br>
  ・[Shibboleth 属性と WEKO3 属性値のマッピング操作](#Shibboleth属性とWEKO3属性値のマッピング操作)<br>
  ・[ブロックユーザーの管理](#ブロックユーザーの管理)

- > 利用方法

  【Administration \> 設定（Setting） \> Shibboleth 画面】にて操作を行う。

  <!-- - 図 1 管理画面：Shibboleth<br>
    <img src="media/media/image13.PNG"> -->

- > 利用可能なロール

<table>
<thead>
<tr class="header">
<th>ロール</th>
<th>システム<br />
管理者</th>
<th>リポジトリ<br />
管理者</th>
<th>コミュニティ<br />
管理者</th>
<th>登録ユーザー</th>
<th>一般ユーザー</th>
<th>ゲスト<br />
(未ログイン)</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>利用可否</td>
<td>○</td>
<td>○</td>
<td></td>
<td></td>
<td></td>
<td></td>
</tr>
</tbody>
</table>

<!-- end list -->

### シボレスユーザーの許可拒否を設定

- > 機能内容

  - 画面には以下の選択肢を持つラジオボタンがあり、現在の許可/拒否設定を反映して表示される

    <!-- 図 2<br>
    <img src="media/media/image14.PNG"> -->

    - 「Shibboleth を有効にする」(Enable Shibboleth Authentication)

      - シボレスユーザーを許可とし、「Shibboleth User」ボタンをログイン画面に表示させる設定

    - 「Shibboleth を無効にする」（Disable Shibboleth Authentication）

      - シボレスユーザーを拒否とし、「Shibboleth User」ボタンをログイン画面に表示させない設定

  - 右下部の［保存（Save）］ボタンを押すと、設定内容を保存し、以下のメッセージを画面上部に表示する  
    JP：「Shibboleth 設定を更新しました」  
    EN：「Updated Shibboleth settings」

<!-- end list -->

- > 関連モジュール

<!-- end list -->

- weko_accounts

<!-- end list -->

- > 処理概要

<!-- end list -->

- 画面表示時に、weko_accounts.admin.ShibSettingView.index メソッドを GET で呼び出して、instance.cfg または weko-accounts で以下のコンフィグから Shibboleth の許可設定を読み込む。両方で設定されている場合、instance.cfg の設定が優先される。また、画面で設定を変更した場合は、その変更が最優先される。

  - パス（instance.cfg）：  
    <https://github.com/RCOSDP/weko/blob/v0.9.22/scripts/instance.cfg#L436>

  - パス（config.py）：  
    <https://github.com/RCOSDP/weko/blob/v0.9.22/modules/weko-accounts/weko_accounts/config.py#L29>

  - 設定キー：WEKO_ACCOUNTS_SHIB_LOGIN_ENABLED

- ［保存（Save）］ボタンを押すと、weko_accounts.admin.ShibSettingView.index メソッドを POST で呼び出して、以下のようにしてコンテキストに設定を保存する。

> \_app = LocalProxy(lambda: current_app.extensions\['weko-admin'\].app)
>
> ※上記は ShibSettingView クラスの外で定義
>
> shib_flg = request.form.get('shibbolethRadios', '0')
>
> if shib_flg == '1':
>
> \_app.config\['WEKO_ACCOUNTS_SHIB_LOGIN_ENABLED'\] = True
>
> else:
>
> \_app.config\['WEKO_ACCOUNTS_SHIB_LOGIN_ENABLED'\] = False

### デフォルトロールの設定

- > 機能内容

  - > [既定のロール]ではシステム利用者のデフォルトロールを設定することができる

    <!-- 図 3<br>
    <img src="media/media/image15.PNG"> -->

    - 候補として選択できるロールは以下の 5 種類
      - System Administrator(システム管理者)
      - Repository Administrator(リポジトリ管理者)
      - Community Administrator(コミュニティ管理者)
      - Contoributer(登録ユーザー)
      - ロール無(一般ユーザー)

  - > システム利用者は以下のように分類されており、それぞれのデフォルトロールを変更することができる

    - [学認 IdP]
      - 学認 IdP からログインしたシステム利用者のデフォルトロールを設定できる<br>
        初期値は「Contoributer(登録ユーザー)」
    - [機関外の Orthros]
      - 機関外の Orthros からログインしたシステム利用者のデフォルトロールを設定できる<br>
        初期値は「Community Administrator(コミュニティ管理者)」<br>
        ※[機関内の Orthros]からログインしたシステム利用者には「Repository Administrator(リポジトリ管理者)」が付与される
    - [上記以外の IdP]
      - 上記以外の IdP からログインしたシステム利用者のデフォルトロールを設定できる<br>
        初期値は「ロール無(一般ユーザー)」

<!-- end list -->

- > 関連モジュール

<!-- end list -->

- weko_accounts

<!-- end list -->

- > 処理概要

  画面表示時に weko_accounts.admin.ShibSettingView.index メソッドを GET で呼び出して、weko-accounts で以下のコンフィグからデフォルトロール設定を読み込む。また、画面で設定を変更した場合は、その変更が最優先される。

  - パス（config.py）：  
    <https://github.com/RCOSDP/weko/blob/v0.9.22/modules/weko-accounts/weko_accounts/config.py#L77>

  - 設定キー：WEKO_ACCOUNTS_SHIB_ROLE_RELATION

- ［保存（Save）］ボタンを押すと、weko_accounts.admin.ShibSettingView.index メソッドを POST で呼び出して、コンテキストに設定を保存する。

<!-- end list -->

### Shibboleth 属性と WEKO3 属性値のマッピング操作

- > 機能内容

  - > [属性マッピング]では Shibboleth 属性と WEKO3 属性値のマッピングを行うことができる

    <!-- 図 4<br>
    <img src="media/media/image16.PNG"> -->

    - 設定を行えるのは以下の 4 項目で、それぞれ有効／無効(True/False)と WEKO3 の属性値を変更することができる

      - SHIB_ATTR_EPPN

      - SHIB_ATTR_ROLE_AUTHORITY_NAME

      - SHIB_ATTR_MAIL

      - SHIB_ATTR_USER_NAME

<!-- end list -->

- > 関連モジュール

<!-- end list -->

- weko_accounts

<!-- end list -->

- > 処理概要

  画面表示時に、weko_accounts.admin.ShibSettingView.index メソッドを GET で呼び出して、instance.cfg または weko-accounts で以下のコンフィグから Shibboleth 属性と WEKO3 属性値のマッピング設定を読み込む。両方で設定されている場合、instance.cfg の設定が優先される。また、画面で設定を変更した場合は、その変更が最優先される。

  - パス（instance.cfg）：  
    <https://github.com/RCOSDP/weko/blob/v0.9.22/scripts/instance.cfg#L442>

  - パス（config.py）：  
    <https://github.com/RCOSDP/weko/blob/v0.9.22/modules/weko-accounts/weko_accounts/config.py#L64>

  - 設定キー：WEKO_ACCOUNTS_SSO_ATTRIBUTE_MAP

- ［保存（Save）］ボタンを押すと、weko_accounts.admin.ShibSettingView.index メソッドを POST で呼び出して、コンテキストに設定を保存する。

<!-- end list -->

### ブロックユーザーの管理

- > 機能内容

  - > [ブロックユーザー]ではあらかじめログインをブロックしたいシステム利用者の ePPN を登録しておくことができる

    <!-- 図 5<br>
    <img src="media/media/image17.PNG"> -->

    - システム利用者が WEKO3 アカウントを未所持の場合、アカウント作成自体ブロックする
    - ePPN はワイルドカードでの指定も可能で、特定の機関からのシステム利用者を丸ごとブロックすることも可能
      - ワイルドカードに指定された機関の中にすでに WEKO3 アカウントの所持者の ePPN が含まれていた場合でもログインブロックの対象として登録可能<br>
    - WEKO3 のアカウント所持者をブロックする場合、［保存（Save）］ボタンを押した際にアラートを表示した上でブロックの一覧に追加する

<!-- end list -->

- > 関連モジュール

<!-- end list -->

- weko_accounts

<!-- end list -->

- > 処理概要

<!-- end list -->

- > 更新履歴

<table>
<thead>
<tr class="header">
<th>日付</th>
<th>GitHubコミットID</th>
<th>更新内容</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><blockquote>
<p>2023/08/31</p>
</blockquote></td>
<td>353ba1deb094af5056a58bb40f07596b8e95a562</td>
<td>初版作成</td>
</tr>
<tr class="even">
<td><blockquote>
<p>2025/02/-</p>
</blockquote></td>
<td></td>
<td>Shibboleth 管理画面を追加</td>
</tr>
</tbody>
</table>
