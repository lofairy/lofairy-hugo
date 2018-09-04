---
title: "Git submodule : 導入其他 repository 並保持互相獨立"
date: 2018-09-04T10:50:00+08:00
draft: false
tags: [
    "submodule",
]
categories: [
    "GIT",
]
---
## 我們經常遇到這樣的問題
在開發的 repo 叫 dev；另一個 repo 叫 util。我將 util 導入 dev 當作函式庫。
我想要 git 處理兩個 repos 是獨立的項目，而非 util 的程式碼被 copy 到 dev repo。

git submodule 可以達成上面的需求。它允許你在 dev 加入 util，當你對 dev 做 commit 時，並不會將 util 的程式碼當作 dev 的一部分。git 只會紀錄 util 的「版號」。

![](https://imgur.com/QITwnYG.jpg)

<!-- more -->

## 怎麼運作的?
本章節內容是基於[這裡](http://dan.mccloy.info/2015/06/11/Git-submodules/)修改的，看得懂英文的朋友建議看這一篇。

1. **建立兩個 repos，並推上 github**。在開發的 repo 叫 dev；另一個 repo 叫 util，它在 dev 的裡面。

2. **建立 submodule**。在 dev 的根目錄，使用 `git submodule add git@github.com:lofairy/util.git` 建立submodule關聯。這時產生的 submodule 是 utils HEAD 的「斷頭狀態」，util 並不真的存在於 dev branch，在 util 所做的任何異動也不會被 dev 追蹤。

3. **在 dev repos 加入以下設定**：
    - `git config status.submodulesummary 1` - 執行 `git status` 內容會包含 submodule 的改變。
    - `git config diff.submodule log` - 執行 `git diff` 會顯示 submodule 的 commit 訊息。

4. **把 submodule 放在 branch 上**。對 submodule 的資料夾做 checkout就行了，實際執行就是 `cd util; git checkout master`。現在你對 util 的改變能夠追蹤了。如果沒有執行這一步，你對 util 做的改變將只是個快照，沒有其他 node 可以參考，也不會顯示在 github 的 commit 裡。

5. **決定 local submodule 的狀態**。submodule 雖然依附在 dev，並且通常情況下只會跟著 dev 的 repo 一起改變，但協作者仍有可能會從 upstream push 異動。你可以用 `cd util; git fetch; git pull origin/magster` 進入 submodule 去做更新。或者是從 dev repo 用 `git submodule update --remote --merge` 將 remote 的異動以 merge 的方式合併進 local submodule。

`--merge` 會是必要的參數，如果少了它，你 local submodule 異動 將會存在於一個沒父沒母的「快照」分支。

以下是對照圖：
with `--merge`
![](https://imgur.com/xwD7ies.jpg)
without `--merge`
![](https://imgur.com/YUfMwGN.jpg)

6. **PUSH**。一般的 `git push` 不會 push submodule 的異動。你可以選擇用 `git push --recurse-submodules=check` 在 push dev 前檢查是否有 submodule 異動沒有 push (如果有，git push 就會警告你並停止繼續動作)，或是用 `git push --recurse-submodules=on-demand` 讓 git 幫你在 push dev 前 自動 push submodule 異動。

## 如何移除 submodule
移除的方法稍微麻煩了點，以下是由 stack overflow 的 [這篇](https://stackoverflow.com/questions/1260748/how-do-i-remove-a-submodule) 文章分享的方法：
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

P.S.當你在 repo add submodule 後，會增加一些檔案跟設定：
- `.git/module/(submodule name)` 儲存了完整的 submodule repo 內容。
- `.git/config` 會儲存包含目前 repo所擁有的 submodule 資訊
- `.gitsubmodule` 會儲存目前 repo所擁有的 submodule 資訊

## 缺點很明顯
1. submodule 真的很容易出錯，除非你是個 git 熟手，不然在使用上導致斷頭的案例比比皆是。
2. 一旦你的異動斷了頭，並且 push 到某個快照分支，就沒有補救的機會了，你得手動把異動在寫一次。
3. **多人協作的開發環境不應該使用它**。一人出錯，大家升天。

## git subtree 是另一個選擇
git submodule 是 git 以前官方推薦管理子項目的功能。從 git 1.5.2 開始，git 推薦使用 git subtree，雖然不具備依賴功能，卻能達到我們的要求。等我研究完後，會再發一篇文說明。

---
## Reference:
> - [Git submodules](http://dan.mccloy.info/2015/06/11/Git-submodules/)
> - [【Git】子模块：一个仓库包含另一个仓库](https://www.jianshu.com/p/491609b1c426)
> - [How do I remove a submodule?](https://stackoverflow.com/questions/1260748/how-do-i-remove-a-submodule)