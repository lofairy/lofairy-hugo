---
title: "Git submodule : 導入其他 repository 並保持互相獨立"
date: 2018-09-04T10:50:00+08:00
draft: false
tags: [
    "git",
    "submodule",
]
categories: [
    "GIT",
]
---
## 我們經常遇到這樣的問題
我有一個專案 repo 叫 dev；另一個 repo 叫 util。我想將 util 導入 dev 當作函式庫。
在以前，我是用 ide 的功能來導入其他專案到另一個專案。
那我能不能用 git 幫我達成這個需求呢？答案是：YES.

git submodule 允許你在 dev 加入 util，並且兩個專案的 commit 是獨立的。當 util 有新的 commit，dev 的 commit 紀錄的是 util 的 commit id。

![](https://imgur.com/QITwnYG.jpg)

## 我認為的優點
- parent 專案只會紀錄 submodule 的 commit id，放在 repo 的程式不包含 submodule 的程式碼。
- parent 與 submodule 專案的 commit 獨立，歷史紀錄明確分立。

## 我認為的缺點
- 操作不當可能會導致 local submodule 與 remote 脫離依賴，沒有辦法追朔的 commit 會很麻煩。
- submodule 會有額外的 metadata，像是 `.gitmodule`。

<!-- more -->
## 怎麼運作的?
本章節內容是基於[這裡](http://dan.mccloy.info/2015/06/11/Git-submodules/)修改的，看得懂英文的朋友建議看這一篇。

> 範例：我有一個專案 repo 叫 dev；另一個 repo 叫 util。我想將 util 導入 dev 當作函式庫。

1. **建立兩個 repos，並推上 github**. 在開發的 repo 叫 dev；另一個 repo 叫 util，它在 dev 的裡面。

2. **建立 submodule**. 在 dev 的根目錄，使用 `git submodule add git@github.com:lofairy/util.git` 建立submodule關聯。這時產生的 submodule 是 utils HEAD 的「斷頭狀態」，util 並不真的存在於 dev branch，在 util 所做的任何 commit 也不會被 dev 追蹤。

3. **在 dev repos 加入以下設定**：
    - `git config status.submodulesummary 1` - 執行 `git status` 內容會包含 submodule 的改變。
    - `git config diff.submodule log` - 執行 `git diff` 會顯示 submodule 的 commit 訊息。

4. **將 local submodule 與 repo 的 branch 綁定依賴**. 對 submodule 的資料夾做 checkout就行了，實際執行就是 `cd util; git checkout master`。現在你對 util 的改變能夠追蹤了。如果沒有執行這一步，你對 util 做的改變將只是個沒有任何依賴的「快照」，push 後不會顯示在 github 的 commit 裡。

> 2, 4 步驟可以用 `git submodule add -b master git@github.com:lofairy/util.git` 整合起來。

5. **修改 submodule**：
    - **update**。最簡單的方法就是 `cd util; git fetch; git pull origin/magster` 進入 submodule 去做更新；或也可以 `git submodule update --remote --merge` 使用submodule 的方式來更新。
    > `--merge` 會是必要的參數，如果少了它，local submodule 將會產並切換到一個沒父沒母的 commit id。
    
    - **commit**。記得修改完後要 commit，不然 dev 會不知道 util 有被修改過。

submodule 雖然依附在 dev，並且通常情況下只會跟著 dev 的專案 一起改變，但協作者仍有可能會從 upstream push commit。你可以用 `cd util; git fetch; git pull origin/magster` 進入 submodule 去做更新。或者是從 dev 專案目錄用 `git submodule update --remote --merge` 將 repo 的 commit 以 merge 的方式合併進 local submodule。

以下是對照圖：
with `--merge`
![](https://imgur.com/xwD7ies.jpg)

without `--merge`
![](https://imgur.com/YUfMwGN.jpg)

6. **PUSH**. 一般的 `git push` 不會 push submodule 的 commit，需要加上特別的參數。
- `git push --recurse-submodules=check` 在 push dev 前檢查是否有 submodule commit 沒有 push (如果有，git push 就會警告你並停止動作)
- `git push --recurse-submodules=on-demand` 讓 git 幫你在 push dev 前自動 push submodule commit。

## 如何移除 submodule
移除的方法稍微麻煩了點，以下是由 [stack overflow](https://stackoverflow.com/questions/1260748/how-do-i-remove-a-submodule) 文章分享的方法：
```
1. Delete the relevant section from the .gitmodules file.
2. Stage the .gitmodules changes git add .gitmodules
3. Delete the relevant section from .git/config.
4. Run git rm --cached path_to_submodule (no trailing slash).
5. Run rm -rf .git/modules/path_to_submodule
6. Commit git commit -m "Removed submodule <name>"
7. Delete the now untracked submodule files
8. rm -rf path_to_submodule
```

以上面的範例來說：

- run `git submodule deinit util -f` 清除 `.git/config` 與 submodule (util)) 相- 關段落，並清除 submodule 資料夾的程式。如果要一次清除，將 util 改成 --all 即可。
- 確認 .gitmodules 與 .git/config 與 submodule (util) 相關的段落是否還存在。如果是，移除他們。
- stash .gitmodules.
- run `git rm -rf --cached util`
- run `git commit -m "remove submodule util"`
- run `rm -rf util`
- run `rm -rf .git/modules/util`


P.S.當你在 repo add submodule 後，會增加一些檔案跟設定：
- `.git/module/(submodule name)` 儲存了完整的 submodule repo 內容。
- `.git/config` 會儲存包含目前 repo所擁有的 submodule 資訊
- `.gitsubmodule` 會儲存目前 repo所擁有的 submodule 資訊

## git subtree 是另一個選擇
git submodule 是 git 以前官方推薦管理子項目的功能。從 git 1.5.2 開始，git 推薦使用 git subtree，雖然不具備依賴功能，卻能達到我們的要求。等我研究完後，會再發一篇文分享。

---
## Reference:
> - [Git submodules](http://dan.mccloy.info/2015/06/11/Git-submodules/)
> - [【Git】子模块：一个仓库包含另一个仓库](https://www.jianshu.com/p/491609b1c426)
> - [How do I remove a submodule?](https://stackoverflow.com/questions/1260748/how-do-i-remove-a-submodule)