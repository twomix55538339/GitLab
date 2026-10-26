# gitlab-ci-local

https://zenn.dev/boy_techblog/articles/gitlab-ci-local

```
curl -L https://github.com/firecow/gitlab-ci-local/releases/latest/download/gitlab-ci-local-windows-amd64.zip -o gcl.zip && unzip -o gcl.zip -d /c/Program\ Files/Git/mingw64/bin && rm gcl.zip
```

下記のエラーが発生しました。
```
curl -L https://github.com/firecow/gitlab-ci-local/releases/latest/download/gitlab-ci-local-windows-amd64.zip -o gcl.zip && unzip -o gcl.zip -d /c/Program\ Files/Git/mingw64/bin && rm gcl.zip
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 53.3M  100 53.3M    0     0  10.0M      0  0:00:05  0:00:05 --:--:-- 11.8M
Archive:  gcl.zip
checkdir:  cannot create extraction directory: /c/Program Files/Git/mingw64/bin
           No such file or directory
```

エラーの原因はパスではなく、WSLのLinuxパスとWindowsパスが混ざっているためです。

実行したコマンド:

```
Shell
unzip -o gcl.zip -d /c/Program\ Files/Git/mingw64/bin
```
WSLでは C: ドライブは /mnt/c にマウントされます。

そのため、

```
Plain Text
/c/Program Files/Git/mingw64/bin
```
ではなく、
```
Plain Text
/mnt/c/Program Files/Git/mingw64/bin
```
が正しいです。

まず Git のインストール先が存在するか確認しましょう。
```
Shell
ls "/mnt/c/Program Files/Git/mingw64/bin"
```
ファイル一覧が表示されればOKです。

その後、管理者権限が必要な可能性があるので、先に unzip をインストールし、
```
Shell
sudo apt update
sudo apt install unzip
```
その後、
```
Shell
unzip -o gcl.zip -d "/mnt/c/Program Files/Git/mingw64/bin"
```
を実行してください。

もし書き込み権限エラーになる場合は、Git for Windows のフォルダが管理者権限を要求している可能性があります。その場合は一旦ユーザー配下に展開します。
```
Shell
mkdir ~/gcl
unzip -o gcl.zip -d ~/gcl
ls ~/gcl
```
ちなみに最近の gitlab-ci-local のインストール方法なら、Windows上で直接以下を実行する方が簡単です。
```
PowerShell
winget install firecow.gitlab-ci-local
```

```
ls "/mnt/c/Program Files/Git/mingw64/bin"
ls: cannot access '/mnt/c/Program Files/Git/mingw64/bin': No such file or directory
```
そのメッセージなら、Git for Windows がその場所にインストールされていないようです。

まず Git のインストール場所を確認しましょう。

WSL で次を実行してください。
```
Shell
ls "/mnt/c/Program Files"
```
その中に Git があるか確認します。

ある場合:
```
Shell
ls "/mnt/c/Program Files/Git"
```
ない場合は、
```
Shell
ls "/mnt/c/Program Files (x86)"
```
も確認してください。

もっと簡単な方法は、Windows 側で PowerShell を開いて、
```
PowerShell
where git
```
を実行することです。

例えば次のような結果になります。
```
Plain Text
C:\Program Files\Git\cmd\git.exe
```
または
```
Plain Text
C:\Users\User\AppData\Local\Programs\Git\cmd\git.exe
```
そのパスが分かれば、gitlab-ci-local の配置先を正しく指定できます。

なお、最近の GitLab CI ローカル実行なら Git のフォルダへ手動コピーせず、PowerShell で
```
PowerShell
winget install firecow.gitlab-ci-local
```
の方が簡単です。

まずは次のどちらかの結果を貼ってください。
```
PowerShell
where git
```
または
```
Shell
ls "/mnt/c/Program Files"
```
そうすれば正しい配置先を特定できます。
```
ls "/mnt/c/Program Files/Git"
LICENSE.txt        bin  dev  git-bash.exe  tmp     unins000.dat  unins000.msg
ReleaseNotes.html  cmd  etc  git-cmd.exe   ucrt64  unins000.exe  usr
```
Git がありましたね。
```
/mnt/c/Program Files/Git
```

ただし、先ほど指定していた
```
/mnt/c/Program Files/Git/mingw64/bin
```
ではなく、この Git の構成だと ucrt64 が存在しています。

まず実際に bin フォルダがどこにあるか確認してください。
```
Shell
ls "/mnt/c/Program Files/Git/usr/bin"
```
または
```
Shell
ls "/mnt/c/Program Files/Git/bin"
```
もし gitlab-ci-local.exe を Git Bash から使いたいだけなら、Git のフォルダへコピーするよりも Windows のユーザーフォルダに配置して PATH を通す方が安全です。

例えば、
```
Shell
mkdir -p ~/bin
unzip -o gcl.zip -d ~/bin
ls ~/bin
```
で展開できます。

なお、今回のエラーの直接原因は
```
Plain Text
cannot create extraction directory: /c/Program Files/Git/mingw64/bin
```
であり、
```
Plain Text
/c/...
```
というパスが存在しなかったことです。WSLでは Windows の C ドライブは
```
Plain Text
/mnt/c/...
```
になります。

また、ダウンロードした ZIP は削除されておらず残っているはずなので、
```
Shell
ls -lh gcl.zip
```
で確認できます。

GitLab CI のローカル実行が目的なら、次に
```
Shell
file gcl.zip
```
または
```
Shell
unzip -l gcl.zip | head
```
を実行して、ZIP 内のファイル名を確認しましょう。どの実行ファイルが入っているか確認して、最適な配置先を案内できます。

あなたの Git for Windows には mingw64 フォルダがなく、ucrt64 構成になっています。

つまり、ネットの記事の
```
Shell
/Program Files/Git/mingw64/bin
```
はあなたの環境には存在しません。

代わりに存在するのは
```
Plain Text
C:\Program Files\Git\usr\bin
C:\Program Files\Git\bin
```
です。

また、gitlab-ci-local は単独の実行ファイルなので、Gitのフォルダへコピーする必要はありません。

例えば Windows のユーザー環境用の bin を作る方が安全です。
```
Shell
mkdir -p ~/bin
unzip -o gcl.zip -d ~/bin
```
確認:
```
Shell
ls ~/bin
```
gitlab-ci-local.exe または類似のファイルが見えるはずです。

もし Windows の Git Bash から使いたいなら、
```
Shell
unzip -o gcl.zip -d "/mnt/c/Program Files/Git/usr/bin"
```
でも配置できますが、Program Files 配下は権限の問題が出やすいです。

まず ZIP の中身を確認しましょう。
```
Shell
unzip -l gcl.zip
```
その結果を貼ってください。中に入っている実行ファイル名を見て、次に実行すべきコマンドを具体的に案内できます。

ちなみに最初の目的だった sudo apt install unzip は、Git for Windows に既に unzip.exe が入っているので、今の作業だけなら Ubuntu 側に unzip をインストールしなくても進められています。パスワード再設定は後回しでも大丈夫です。

unzip -l gcl.zip
Archive:  gcl.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
117427200  2026-10-25 04:01   gitlab-ci-local.exe
---------                     -------
117427200                     1 file

ZIP の中身はこれだけですね。
```
Plain Text
gitlab-ci-local.exe
```
つまり、展開先にその EXE を置くだけで使えます。

現在の環境なら、Git for Windows の実行ファイルがある場所にコピーするのが一番簡単です。
```
Shell
unzip -o gcl.zip -d "/mnt/c/Program Files/Git/usr/bin"
```
成功したら確認:
```
Shell
ls "/mnt/c/Program Files/Git/usr/bin/gitlab-ci-local.exe"
```
その後、Git Bash または新しいターミナルを開いて
```
Shell
gitlab-ci-local --version
```
または
```
Shell
gitlab-ci-local
```
を実行してください。

もし権限エラーが出る場合は、Program Files 配下への書き込みが制限されているので、代わりに Windows ユーザーの PATH 配下へ置きます。
```
Shell
mkdir -p "/mnt/c/Users/User/bin"
unzip -o gcl.zip -d "/mnt/c/Users/User/bin"
```
ちなみに、あなたの元のコマンドは
```
Shell
-d /c/Program\ Files/Git/mingw64/bin
```
の部分が古い記事向けでした。

あなたの Git は
```
Plain Text
C:\Program Files\Git\usr\bin
```
と
```
Plain Text
C:\Program Files\Git\bin
```
しか存在しないため、mingw64/bin が見つからずエラーになっていました。

次は以下を実行して結果を見せてください。
```
Shell
unzip -o gcl.zip -d "/mnt/c/Program Files/Git/usr/bin"
```
エラーが出るか、正常に展開できるかで次の手順が決まります。
```
unzip -o gcl.zip -d "/mnt/c/Program Files/Git/usr/bin"
Archive:  gcl.zip
error:  cannot create /mnt/c/Program Files/Git/usr/bin/gitlab-ci-local.exe
        Permission denied
```
権限の問題です。

C:\Program Files\Git\... は Windows の保護されたフォルダーなので、WSL の通常ユーザー ia では書き込めません。

今は gitlab-ci-local を動かすことが目的なので、まずは自分のホームに展開しましょう。
```
Shell
mkdir -p ~/bin
unzip -o gcl.zip -d ~/bin
```
確認:
```
Shell
ls ~/bin
```
おそらく
```
Plain Text
gitlab-ci-local.exe
```
が表示されます。

実行確認:
```
Shell
~/bin/gitlab-ci-local.exe --version
```
または
```
Shell
~/bin/gitlab-ci-local.exe --help
```
もし毎回フルパスを書きたくなければ、
```
Shell
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```
を実行します。

その後、
```
Shell
gitlab-ci-local.exe --version
```
で実行できます。

なお、あなたは先ほど sudo パスワードを忘れていると言っていたので、sudo apt install unzip や Program Files へのコピーを行うには、先にパスワード再設定をした方がよいです。

PowerShell から:
```
PowerShell
wsl -d Ubuntu -u root
```
で root 起動し、
```
Shell
passwd ia
```
でパスワードを再設定しておくことをおすすめします。

まずは以下を実行してみてください。
```
Shell
mkdir -p ~/bin
unzip -o gcl.zip -d ~/bin
~/bin/gitlab-ci-local.exe --version
```
これで gitlab-ci-local 自体が起動するか確認できます。
```
~/bin/gitlab-ci-local.exe --version
4.77.0
```
と表示されたので、gitlab-ci-local のインストールは成功しています。

つまり現時点で：

✅ ZIP のダウンロード成功
 ✅ 展開成功
 ✅ 実行ファイル起動成功
 ✅ バージョン確認成功

次は GitLab のプロジェクトフォルダへ移動して、
```
gitlab-ci-local.exe
```
または
```
Shell
gitlab-ci-local.exe --list
```
を実行してみてください。

例えば、
```
Shell
cd /mnt/c/path/to/your/gitlab/project
gitlab-ci-local.exe --list
```
で .gitlab-ci.yml に定義されているジョブ一覧が表示されます。

特定のジョブを実行する場合は、
```
Shell
gitlab-ci-local.exe build_job
```
のようにジョブ名を指定します。

ちなみに、あなたが開いている GitLab のジョブ画面を見ると、ローカルで .gitlab-ci.yml のデバッグをしたかったようなので、次は

```
Shell
cd GitLabプロジェクト
gitlab-ci-local.exe --list
```

を実行して、ジョブが認識されるか確認する段階です。