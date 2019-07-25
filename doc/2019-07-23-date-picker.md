# 開發筆記｜自己動手刻一個萬年曆與日期選擇器

平常在網頁前端開發工作中，遇到需要引用日期選擇器（date picker）的地方都會直接使用套件，只要簡單的安裝與載入就可以完成。

由於最近正如火如荼的在尋找新工作，幾個禮拜前接到一份面談前的作業，規定要做一個萬年曆與日期選擇器，而且要完全自己動手刻，不能使用任何套件。於是在兩三天的趕工下擠出了成品：

- [Live Demo](https://dezchuang.github.io/vue-date-picker)

以下來記錄一下開發筆記。

## 基礎建設

由於題目要求要設計可重用的模組，且建議可以使用框架與開發工具，所以選擇用最近比較熟悉的 Vue 來開發與管理程式碼。首先用 Vue CLI 快速的創建一個專案：

```
vue create vue-date-picker
```

選擇手動設定（Manually select features）後挑選了 Babel、Router、CSS Pre-processors、Linter / Formatter 這些 feature，剩下再針對這些 feature 設定就起好一個新專案了。

這邊的 linting rule 跟我平常使用的不太一樣，可能是有選錯設定，所以手動調整了一下，更改 `.eslintrc.js` 這個檔案規則，讓後面開發更順暢。剩下就是整理 component 的檔案結構，就可以開始動手開發功能了！

## Task 1. 萬年曆

要手刻一個日期選擇器，首先就得先要做出一個萬年曆，而這正是整份作業比較麻煩的部分。根據指定規格加上參考了一些既有的套件像是「[vuejs-datepicker](https://github.com/charliekassel/vuejs-datepicker)」、「[ElementUI 的 DatePicker](https://element.eleme.io/#/zh-CN/component/date-picker)」，大概列出了以下的實作概觀：

- 資料處理
  - 計算特定月份有幾天
  - 計算特定日期為星期幾
  - 重新整理資料結構，切開日期相關資料關係
- 功能實作
  - 利用資料排出當日的月曆
  - 點擊左右箭頭切月份，秀出對應的月曆
  - 每個月曆秀出六行（包含上個月末與下個月初）
  - 點擊月份切到年曆、點擊年份切到十年曆
  - 點擊年份切到年曆、點擊月份切到月曆
  - 根據當前曆種，點擊左右箭頭用來切換月曆、年曆、十年曆
- 排版樣式調整
  - 不在這個月份的「日期」、十年份的「年」以灰字顯示
  - 當前日期、月份、年份，以紅字顯示
  - 被選中的項目以紅圈背景白字顯示
- 其他後續調整
  - 選取上下個月日期灰字可以切過去對應月曆
  - 實作 pros 及 demo page

### 資料處理

第一步從資料開始準備起。從月曆切入，那我需要的就是「這個月的天數」及「這個月每一天怎麼排」。

先從最簡單的「特定月份有幾天」開始算起，原本以為就是單純的判斷是不是閏年的問題，那除以 4 判斷就好拉，但仔細查才知道原來閏年規則沒這麼單純，實際上是像這樣：

- 西元年不可被 4 整除，平年。
- 西元年可被 4 整除，且不可被 100 整除，閏年。
- 西元年可被 100 整除，且不可被 400 整除，平年
- 西元年可被 400 整除，閏年。

真是長知識了！於是應用這個規則寫成 method 如下：

```
countDaysInMonth(year, month) {
  return /3|5|8|10/.test(month) ? 30 : month === 1 ? ((!(year % 4) && year % 100) || !(year % 400) ? 29 : 28) : 31
}
```

這裡我的月份對應是從 0 開始，也就是根據年份，這一串判斷式大概就是 4、6、9、11 月是 30 天、2 月依照上述閏年規則、其他月份都是 31 天。

接下來另一個問題是「如何找到某一天顯示在月曆中的哪個地方」，這問題換個角度想就是「如何決定某一日是星期幾」。一開始參考了規格中提到可以參考的[演算法文章](https://calendars.wikia.org/wiki/Calculating_the_day_of_the_week)，文章從最基本的概念開始講解到如何推導出演算法。

看完了對應月份的規則後，要讀完全部概念又自己從頭實作會太久，這樣應該沒辦法有效率地完成。於是在另外的搜集資訊後，找到了一般在實作萬年曆時，大部分演算法會使用的蔡勒公式（[Zeller’s algorithm](https://en.wikipedia.org/wiki/Zeller%27s_congruence)）。

> 啊，原來數學家都已經整理好了呢，只要帶公式就好了。

於是將公式內容寫成一個 method 專門處理「傳入特定日期回傳星期幾」：

```
zellerCongruence(year, month, day) {
  // 蔡勒公式中 1, 2 月視為前一年的 13, 14 月
  if (month === 1 || month === 2) {
    month += 12
    year -= 1
  }
  const c = Math.floor(year / 100) // 年份前兩位數
  const y = year % 100 // 年份後兩位數
  const m = month
  const d = day
  let w = 0
  // TODO: 1582.10.15 以後改曆，目前尚未處理 1582.10.4 以前公式
  w = y + Math.floor(y / 4) + Math.floor(c / 4) - 2 * c + Math.floor((26 * (m + 1)) / 10) + d - 1
  if (w < 0) w = (w & (7 + 7)) % 7
  else w = w % 7
  return w
}
```

這邊有個有趣的小知識是西元 1582 年 10 月改曆，所以正確的月曆應該要長這個樣子：

![1582.10 月曆](https://theuijunkie.com/wp-content/uploads/2017/02/october1582-copy.png)
[圖片出處](https://theuijunkie.com/october-5th-october-14th-1582/)

因此 1582.10.04 以前的日期規則不一樣，要用另一個公式處理，但確認了一些常見的 date picker 套件都沒有對這個特例做處理，而且也幾乎不太有機會用到四百多年前的月曆，甚至有些套件的最早的年份只到 1900 年，所以這邊就先不處理。

以上有了  這兩項資料後，就可以組出某個特定月份的月曆資料，這邊利用上述兩個 method 用一個 computed 去準備這份資料：

```
daysInMonth() {
  const arr = []
  const year = this.displayDate.year
  const month = this.displayDate.month
  const currentDayCount = this.countDaysInMonth(year, month)
  const firstDay = this.zellerCongruence(year, month + 1, 1)

  // 依照每個月六行、每行七天規則去排序出月曆資料
  ...

  return arr
}
```

這邊省略的部分寫的稍微醜一些，直接用兩個迴圈依照邏輯先塞出資料，可能可讀性差一些可以再優化。

### 功能實作

這部分相對就比較單純了，依照前面的綱要，將功能一個一個做出來。

從當月月份切入，在 HTML 中利用 table 去呈現月曆資料，再利用資料切換將每個月曆的畫面做出來，進而將年曆、十年曆補上就完成了，這部分及基本上沒有碰到太大的問題。

只有在「日期」資料的處理上，一開始只用同一份資料去控制「畫面顯示」與「選定日期」，但在實作一些像是「選定日期要顯示紅圈」的功能時才發現要切開，所以最後有根據「當前畫面顯示曆種時用來計算的日期」、「目前被選中的日期」、「今天的日期」這三種才能更有彈性的做到後面的功能。

另外在實作選定日期、今天日期樣式切換時，用了比較繁瑣的判斷去檢查，所以寫了很多 `checkSelected*` 和 `checkCurrent*` 這些 method，可能這部份的流程還可以再優化，但目前以完成功能為主先這樣做。

### 其他後續調整

最後依照規格要求，提供兩個 props 操作這個 component：

- date：就是前述的「目前被選中的日期」。
- onSelect：在萬年曆元件上選取日期後，會觸發此事件，提供父元件做相對應需要的處理。

在 demo page 上，Task 1 中只簡單顯示目前被選定的日期，實際應用會在日期選擇器中實作。

## Task 2. 日期選擇器

做完萬年曆元件後，第二部分就簡單一些了，只要將萬年曆元件套用進來，大致上就是像在使用現成套件一樣。

這邊提供使用者兩種操作 date picker 的方式：

- 透過 input 欄位輸入 YYYY-MM-DD 的日期格式
  - 串 input 與 Calendar 元件資料時，利用 `v-model` 綁定 `inputDate` 資料
  - 在按下 enter 時才會去驗證，格式正確才將值傳入 `selectedDate`
  - `selectedDate` 同時也是傳入 Calendar 的 props date，達到可透過輸入跳到該日月曆的效果
- 透過 date picker 選定日期
  - 選定日期後，會觸發 `onSelect` 事件，將父層資料 `selectedDate` 指定為選定日期
  - 將 input 欄位資料 `inputDate` 利用 computed `dateText` 轉換設定為對應字串

由於我在日期資料結構設計上，是用 object 將年月日分開，所以在 Calendar 選定日期後，要將資料轉回 input 欄位另外做了一個字串處理，將日期物件轉成 YYYY-MM-DD 的字串：

```
dateText() {
  const { year, month, day } = this.selectedDate
  const zMonth = ('0' + (month + 1)).slice(-2)
  const zDay = ('0' + day).slice(-2)
  return `${year}-${zMonth}-${zDay}`
}
```

另外其實當初交出這一份作業時，這個 date picker 沒辦法在點開後點擊旁邊空白處關閉，讓我覺得使用體驗不是太好，後來搜尋了一些 [Vue 在這方面的做法](https://medium.com/@Taha_Shashtari/an-easy-way-to-detect-clicks-outside-an-element-in-vue-1b51d43ff634)，發現可以寫成一個自定義的 [directives](https://vuejs.org/v2/guide/custom-directive.html) 來完成這件事：

```
directives: {
  'click-outside': {
    bind(el, { value }) {
      handleOutsideClick = e => {
        const isClickOutside = e.target !== el && !el.contains(e.target)
        if (isClickOutside) value(e)
        e.stopPropagation()
      }
      document.addEventListener('click', handleOutsideClick)
      document.addEventListener('touchstart', handleOutsideClick)
    },
    unbind() {
      document.removeEventListener('click', handleOutsideClick)
      document.removeEventListener('touchstart', handleOutsideClick)
    }
  }
}
```

最後再將 input 欄位與 Calendar 元件包在一個 div 中，在這個 div 上加 `v-click-outside="onClose"` 就可以達成想要的效果了，可喜可賀！

## 後記

第一次自己動手刻萬年曆跟日期選擇器的體驗還是蠻有趣的，要不是剛好有這個機會練習，可能也不會知道關於萬年曆的這些細節，也在寫文章時為了要將功能呈現完整，加上了最後提到的 click outside 的功能，第一次嘗試在 Vue 中自己寫 directives 也是學會一個新玩意，整體而言學到不少東西，齁勝！

## 參考資料

- https://github.com/charliekassel/vuejs-datepicker
- https://element.eleme.io/#/zh-CN/component/date-picker
- https://en.wikipedia.org/wiki/Zeller%27s_congruence
- https://medium.com/@Taha_Shashtari/an-easy-way-to-detect-clicks-outside-an-element-in-vue-1b51d43ff634
