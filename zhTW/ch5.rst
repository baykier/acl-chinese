.. highlight:: cl

第五章：控制流
***************************************************

2.2 節介紹過 Common Lisp 的求值規則，現在你應該很熟悉了。本章的運算子都有一個共同點，就是它們都違反了求值規則。這些運算子讓你決定在程式當中何時要求值。如果普通的函數呼叫是 Lisp 程式的樹葉的話，那這些運算子就是連結樹葉的樹枝。

5.1 區塊(Blocks)
==========================

Common Lisp 有三個構造區塊（block）的基本運算子： ``progn`` 、 ``block`` 以及 ``tagbody`` 。我們已經看過 ``progn`` 了。在 ``progn`` 主體中的表達式會依序求值，並返回最後一個表達式的值：

::

  > (progn
      (format t "a")
      (format t "b")
      (+ 11 12))
  ab
  23

由於只返回最後一個表達式的值，代表著使用 ``progn`` （或任何區塊）涵蓋了副作用。

一個 ``block`` 像是帶有名字及緊急出口的 ``progn`` 。第一個實參應爲符號。這成爲了區塊的名字。在主體中的任何地方，可以停止求值，並通過使用 ``return-from`` 指定區塊的名字，來立即返回數值：

::

  > (block head
      (format t "Here we go.")
      (return-from head 'idea)
      (format t "We'll never see this."))
  Here we go.
  IDEA

呼叫 ``return-from`` 允許你的程式，從程式的任何地方，突然但優雅地退出。第二個傳給 ``return-from`` 的實參，用來作爲以第一個實參爲名的區塊的返回值。在 ``return-from`` 之後的表達式不會被求值。

也有一個 ``return`` 宏，它把傳入的參數當做封閉區塊 ``nil`` 的返回值：

::

  > (block nil
      (return 27))
  27

許多接受一個表達式主體的 Common Lisp 運算子，皆隱含在一個叫做 ``nil`` 的區塊裡。比如，所有由 ``do`` 構造的迭代函數：

::

  > (dolist (x '(a b c d e))
      (format t "~A " x)
      (if (eql x 'c)
          (return 'done)))
  A B C
  DONE

使用 ``defun`` 定義的函數主體，都隱含在一個與函數同名的區塊，所以你可以：

::

  (defun foo ()
    (return-from foo 27))

在一個顯式或隱式的 ``block`` 外，不論是 ``return-from`` 或 ``return`` 都不會工作。

使用 ``return-from`` ，我們可以寫出一個更好的 ``read-integer`` 版本：

::

	(defun read-integer (str)
	  (let ((accum 0))
	    (dotimes (pos (length str))
	      (let ((i (digit-char-p (char str pos))))
	        (if i
	            (setf accum (+ (* accum 10) i))
	            (return-from read-integer nil))))
	    accum))

68 頁的版本在構造整數之前，需檢查所有的字元。現在兩個步驟可以結合，因爲如果遇到非數字的字元時，我們可以捨棄計算結果。出現在主體的原子（atom）被解讀爲標籤（labels)；把這樣的標籤傳給 ``go`` ，會把控制權交給標籤後的表達式。以下是一個非常醜的程式片段，用來印出一至十的數字：

::

  > (tagbody
      (setf x 0)
      top
        (setf x (+ x 1))
        (format t "~A " x)
        (if (< x 10) (go top)))
  1 2 3 4 5 6 7 8 9 10
  NIL

這個運算子主要用來實現其它的運算子，不是一般會用到的運算子。大多數迭代運算子都隱含在一個 ``tagbody`` ，所以是可能可以在主體裡（雖然很少想要）使用標籤及 ``go`` 。

如何決定要使用哪一種區塊建構子呢（block construct）？幾乎任何時候，你會使用 ``progn`` 。如果你想要突然退出的話，使用 ``block`` 來取代。多數程式設計師永遠不會顯式地使用 ``tagbody`` 。

5.2 語境（Context）
==========================

另一個我們用來區分表達式的運算子是 ``let`` 。它接受一個程式碼主體，但允許我們在主體內設置新變數：

::

  > (let ((x 7)
          (y 2))
      (format t "Number")
      (+ x y))
  Number
  9

一個像是 ``let`` 的運算子，創造出一個新的詞法語境（lexical context）。在這個語境裡有兩個新變數，然而在外部語境的變數也因此變得不可視了。

概念上說，一個 ``let`` 表達式等同於函數呼叫。在 2.14 節證明過，函數可以用名字來引用，也可以通過使用一個 lambda 表達式從字面上來引用。由於 lambda 表達式是函數的名字，我們可以像使用函數名那樣，把 lambda 表達式作爲函數呼叫的第一個實參：

::

  > ((lambda (x) (+ x 1)) 3)
  4

前述的 ``let`` 表達式，實際上等同於：

::

  ((lambda (x y)
     (format t "Number")
     (+ x y))
   7
   2)

如果有關於 ``let`` 的任何問題，應該是如何把責任交給 ``lambda`` ，因爲進入一個 ``let`` 等同於執行一個函數呼叫。

這個模型清楚的告訴我們，由 ``let`` 創造的變數的值，不能依賴其它由同一個 ``let`` 所創造的變數。舉例來說，如果我們試著：

::

  (let ((x 2)
        (y (+ x 1)))
    (+ x y))

在 ``(+ x 1)`` 中的 ``x`` 不是前一行所設置的值，因爲整個表達式等同於：

::

  ((lambda (x y) (+ x y)) 2
                          (+ x 1))

這裡明顯看到 ``(+ x 1)`` 作爲實參傳給函數，不能引用函數內的形參 ``x`` 。

所以如果你真的想要新變數的值，依賴同一個表達式所設立的另一個變數？在這個情況下，使用一個變形版本 ``let*`` ：

::

  > (let* ((x 1)
           (y (+ x 1)))
      (+ x y))
  3

一個 ``let*`` 功能上等同於一系列巢狀的 ``let`` 。這個特別的例子等同於：

::

  (let ((x 1))
    (let ((y (+ x 1)))
      (+ x y)))

``let`` 與 ``let*`` 將變數初始值都設爲 ``nil`` 。``nil`` 爲初始值的變數，不需要依附在列表內:

::

  > (let (x y)
      (list x y))
  (NIL NIL)

``destructuring-bind`` 宏是通用化的 ``let`` 。與其接受單一變數，一個模式 (pattern) ── 一個或多個變數所構成的樹 ── 並將它們與某個實際的樹所對應的部份做綁定。舉例來說：

::

  > (destructuring-bind (w (x y) . z) '(a (b c) d e)
      (list w x y z))
  (A B C (D E))

若給定的樹（第二個實參）沒有與模式匹配（第一個參數）時，會產生錯誤。

5.3 條件 (Conditionals)
===========================

最簡單的條件式是 ``if`` ；其餘的條件式都是基於 ``if`` 所構造的。第二簡單的條件式是 ``when`` ，它接受一個測試表達式（test expression）與一個程式碼主體。若測試表達式求值返回真時，則對主體求值。所以

::

  (when (oddp that)
    (format t "Hmm, that's odd.")
    (+ that 1))

等同於

::

  (if (oddp that)
      (progn
        (format t "Hmm, that's odd.")
        (+ that 1)))

``when`` 的相反是 ``unless`` ；它接受相同的實參，但僅在測試表達式返回假時，才對主體求值。

所有條件式的母體 (從正反兩面看) 是 ``cond`` ， ``cond`` 有兩個新的優點：允許多重條件判斷，與每個條件相關的程式碼隱含在 ``progn`` 裡。 ``cond`` 預期在我們需要使用巢狀 ``if`` 的情況下使用。 舉例來說，這個僞 member 函數

::

  (defun our-member (obj lst)
    (if (atom lst)
        nil
        (if (eql (car lst) obj)
            lst
            (our-member obj (cdr lst)))))

也可以定義成：

::

  (defun our-member (obj lst)
    (cond ((atom lst) nil)
          ((eql (car lst) obj) lst)
          (t (our-member obj (cdr lst)))))

事實上，Common Lisp 實現大概會把 ``cond`` 翻譯成 ``if`` 的形式。

總得來說呢， ``cond`` 接受零個或多個實參。每一個實參必須是一個具有條件式，伴隨著零個或多個表達式的列表。當 ``cond`` 表達式被求值時，測試條件式依序求值，直到某個測試條件式返回真才停止。當返回真時，與其相關聯的表達式會被依序求值，而最後一個返回的數值，會作爲 ``cond`` 的返回值。如果符合的條件式之後沒有表達式的話：

::

  > (cond (99))
  99

則會返回條件式的值。

由於 ``cond`` 子句的 ``t`` 條件永遠成立，通常我們把它放在最後，作爲預設的條件式。如果沒有子句符合時，則 ``cond`` 返回 ``nil`` ，但利用 ``nil`` 作爲返回值是一種很差的風格 (這種問題可能發生的例子，請看 292 頁)。譯註: **Appendix A, unexpected nil** 小節。

當你想要把一個數值與一系列的常數比較時，有 ``case`` 可以用。我們可以使用 ``case`` 來定義一個函數，返回每個月份中的天數：

::

  (defun month-length (mon)
    (case mon
      ((jan mar may jul aug oct dec) 31)
      ((apr jun sept nov) 30)
      (feb (if (leap-year) 29 28))
      (otherwise "unknown month")))

一個 ``case`` 表達式由一個實參開始，此實參會被拿來與每個子句的鍵值做比較。接著是零個或多個子句，每個子句由一個或一串鍵值開始，跟隨著零個或多個表達式。鍵值被視爲常數；它們不會被求值。第一個參數的值被拿來與子句中的鍵值做比較 (使用 ``eql`` )。如果匹配時，子句剩餘的表達式會被求值，並將最後一個求值作爲 ``case`` 的返回值。

預設子句的鍵值可以是 ``t`` 或 ``otherwise`` 。如果沒有子句符合時，或是子句只包含鍵值時，

::

  > (case 99 (99))
  NIL

則 ``case`` 返回 ``nil`` 。

``typecase`` 宏與 ``case`` 相似，除了每個子句中的鍵值應爲型別修飾符 (type specifiers)，以及第一個實參與鍵值比較的函數使用 ``typep`` 而不是 ``eql`` (一個 ``typecase`` 的例子在 107 頁)。 **譯註: 6.5 小節。**

5.4 迭代 (Iteration)
==========================

最基本的迭代運算子是 ``do`` ，在 2.13 小節介紹過。由於 ``do`` 包含了隱式的 ``block`` 及 ``tagbody`` ，我們現在知道是可以在 ``do`` 主體內使用 ``return`` 、 ``return-from`` 以及 ``go`` 。

2.13 節提到 ``do`` 的第一個參數必須是說明變數規格的列表，列表可以是如下形式：

::

  (variable  initial  update)

``initial`` 與 ``update`` 形式是選擇性的。若 ``update`` 形式忽略時，每次迭代時不會更新變數。若 ``initial`` 形式也忽略時，變數會使用 ``nil`` 來初始化。

在 23 頁的例子中（譯註: 2.13 節），

::

  (defun show-squares (start end)
    (do ((i start (+ i 1)))
        ((> i end) 'done)
      (format t "~A ~A~%" i (* i i))))

``update`` 形式引用到由 ``do`` 所創造的變數。一般都是這麼用。如果一個 ``do`` 的 ``update`` 形式，沒有至少引用到一個 ``do`` 創建的變數時，反而很奇怪。

當同時更新超過一個變數時，問題來了，如果一個 ``update`` 形式，引用到一個擁有自己的 ``update`` 形式的變數時，它會被更新呢？或是獲得前一次迭代的值？使用 ``do`` 的話，它獲得後者的值：

::

  > (let ((x 'a))
      (do ((x 1 (+ x 1))
           (y x x))
          ((> x 5))
        (format t "(~A ~A)  " x y)))
  (1 A)  (2 1)  (3 2)  (4 3)  (5 4)
  NIL

每一次迭代時， ``x`` 獲得先前的值，加上一； ``y`` 也獲得 ``x`` 的前一次數值。

但也有一個 ``do*`` ，它有著和 ``let`` 與 ``let*`` 一樣的關係。任何 ``initial`` 或 ``update`` 形式可以參照到前一個子句的變數，並會獲得當下的值：

::

  > (do* ((x 1 (+ x 1))
        (y x x))
       ((> x 5))
    (format t "(~A ~A) " x y))
  (1 1) (2 2) (3 3) (4 4) (5 5)
  NIL

除了 ``do`` 與 ``do*`` 之外，也有幾個特別用途的迭代運算子。要迭代一個列表的元素，我們可以使用 ``dolist`` :

::

  > (dolist (x '(a b c d) 'done)
      (format t "~A " x))
  A B C D
  DONE

當迭代結束時，初始列表內的第三個表達式 (譯註: ``done`` ) ，會被求值並作爲 ``dolist`` 的返回值。預設是 ``nil`` 。

有著同樣的精神的是 ``dotimes`` ，給定某個 ``n`` ，將會從整數 ``0`` ，迭代至 ``n-1`` :

::

  (dotimes (x 5 x)
    (format t "~A " x))
  0 1 2 3 4
  5

``dolist`` 與 ``dotimes`` 初始列表的第三個表達式皆可省略，省略時為 ``nil`` 。注意該表達式可引用到迭代過程中的變數。

（譯註：第三個表達式即上例之 ``x`` ，可以省略，省略時 ``dotimes`` 表達式的回傳值為 ``nil`` ）

.. note::

  do 的重點 (THE POINT OF do)

  在 “The Evolution of Lisp” 裡，Steele 與 Garbriel 陳述了 do 的重點，
  表達的實在太好了，值得整個在這裡引用過來：

  撇開爭論語法不談，有件事要說明的是，在任何一個編程語言中，一個迴圈若一次只能更新一個變數是毫無用處的。
  幾乎在任何情況下，會有一個變數用來產生下個值，而另一個變數用來累積結果。如果迴圈語法只能產生變數，
  那麼累積結果就得藉由賦值語句來“手動”實現…或有其他的副作用。具有多變數的 do 迴圈，體現了產生與累積的本質對稱性，允許可以無副作用地表達迭代過程：

  .. code-block:: cl

      (defun factorial (n)
        (do ((j n (- j 1))
             (f 1 (* j f)))
          ((= j 0) f)))

  當然在 step 形式裡實現所有的實際工作，一個沒有主體的 do 迴圈形式是較不尋常的。

函數 ``mapc`` 和 ``mapcar`` 很像，但不會 ``cons`` 一個新列表作爲返回值，所以使用的唯一理由是爲了副作用。它們比 ``dolist`` 來得靈活，因爲可以同時遍歷多個列表：

::

  > (mapc #'(lambda (x y)
            (format t "~A ~A  " x y))
        '(hip flip slip)
        '(hop flop slop))
  HIP HOP  FLIP FLOP  SLIP SLOP
  (HIP FLIP SLIP)

總是回傳 ``mapc`` 的第二個參數。

5.5 多值 (Multiple Values)
=======================================

曾有人這麼說，爲了要強調函數式編程的重要性，每個 Lisp 表達式都返回一個值。現在事情不是這麼簡單了；在 Common Lisp 裡，一個表達式可以返回零個或多個數值。最多可以返回幾個值取決於各家實現，但至少可以返回 19 個值。

多值允許一個函數返回多件事情的計算結果，而不用構造一個特定的結構。舉例來說，內建的 ``get-decoded-time`` 返回 9 個數值來表示現在的時間：秒，分，時，日期，月，年，天，以及另外兩個數值。

多值也使得查詢函數可以分辨出 ``nil`` 與查詢失敗的情況。這也是爲什麼 ``gethash`` 返回兩個值。因爲它使用第二個數值來指出成功還是失敗，我們可以在雜湊表裡儲存 ``nil`` ，就像我們可以儲存別的數值那樣。

``values`` 函數返回多個數值。它一個不少地返回你作爲數值所傳入的實參：

::

  > (values 'a nil (+ 2 4))
  A
  NIL
  6

如果一個 ``values`` 表達式，是函數主體最後求值的表達式，它所返回的數值變成函數的返回值。多值可以原封不地通過任何數量的返回來傳遞：

::

  > ((lambda () ((lambda () (values 1 2)))))
  1
  2

然而若只預期一個返回值時，第一個之外的值會被捨棄：

::

  > (let ((x (values 1 2)))
      x)
  1

通過不帶實參使用 ``values`` ，是可能不返回值的。在這個情況下，預期一個返回值的話，會獲得 ``nil`` :

::

  > (values)
  > (let ((x (values)))
      x)
  NIL

要接收多個數值，我們使用 ``multiple-value-bind`` :

::

  > (multiple-value-bind (x y z) (values 1 2 3)
      (list x y z))
  (1 2 3)

  > (multiple-value-bind (x y z) (values 1 2)
      (list x y z))
  (1 2 NIL)

如果變數的數量大於數值的數量，剩餘的變數會是 ``nil`` 。如果數值的數量大於變數的數量，多餘的值會被捨棄。所以只想印出時間我們可以這麼寫:

::

  > (multiple-value-bind (s m h) (get-decoded-time)
      (format t "~A:~A:~A" h m s))
  "4:32:13"

你可以藉由 ``multiple-value-call`` 將多值作爲實參傳給第二個函數：

::

  > (multiple-value-call #'+ (values 1 2 3))
  6

還有一個函數是 ``multiple-value-list`` :

::

  > (multiple-value-list (values 'a 'b 'c))
  (A B C)

看起來像是使用 ``#'list`` 作爲第一個參數的來呼叫 ``multiple-value-call`` 。

5.6 中止 (Aborts)
==========================

你可以使用 ``return`` 在任何時候離開一個 ``block`` 。有時候我們想要做更極端的事，在數個函數呼叫裡將控制權轉移回來。要達成這件事，我們使用 ``catch`` 與 ``throw`` 。一個 ``catch`` 表達式接受一個標籤（tag），標籤可以是任何型別的物件，伴隨著一個表達式主體：

::

  (defun super ()
    (catch 'abort
      (sub)
      (format t "We'll never see this.")))

  (defun sub ()
    (throw 'abort 99))

表達式依序求值，就像它們是在 ``progn`` 裡一樣。在這段程式裡的任何地方，一個帶有特定標籤的 ``throw`` 會導致 ``catch`` 表達式直接返回：

::

  > (super)
  99

一個帶有給定標籤的 ``throw`` ，爲了要到達匹配標籤的 ``catch`` ，會將控制權轉移 (因此殺掉進程)給任何有標籤的 ``catch`` 。如果沒有一個 ``catch`` 符合欲匹配的標籤時， ``throw`` 會產生一個錯誤。

呼叫 ``error`` 同時中斷了執行，本來會將控制權轉移到呼叫樹（calling tree）的更高點，取而代之的是，它將控制權轉移給 Lisp 錯誤處理器（error handler）。通常會導致呼叫一個中斷迴圈（break loop）。以下是一個假定的 Common Lisp 實現可能會發生的事情：

::

  > (progn
      (error "Oops!")
      (format t "After the error."))
  Error: Oops!
         Options: :abort, :backtrace
  >>

譯註：2 個 ``>>`` 顯示進入中斷迴圈了。

關於錯誤與狀態的更多訊息，參見 14.6 小節以及附錄 A。

有時候你想要防止程式被 ``throw`` 與 ``error`` 打斷。藉由使用 ``unwind-protect`` ，可以確保像是前述的中斷，不會讓你的程式停在不一致的狀態。一個 ``unwind-protect`` 接受任何數量的實參，並返回第一個實參的值。然而即便是第一個實參的求值被打斷時，剩下的表達式仍會被求值：

::

  > (setf x 1)
  1
  > (catch 'abort
      (unwind-protect
        (throw 'abort 99)
        (setf x 2)))
  99
  > x
  2

在這裡，即便 ``throw`` 將控制權交回監測的 ``catch`` ， ``unwind-protect`` 確保控制權移交時，第二個表達式有被求值。無論何時，一個確切的動作要伴隨著某種清理或重置時， ``unwind-protect`` 可能會派上用場。在 121 頁提到了一個例子。

5.7 範例：日期運算 (Example: Date Arithmetic)
====================================================

在某些應用裡，能夠做日期的加減是很有用的 ── 舉例來說，能夠算出從 1997 年 12 月 17 日，六十天之後是 1998 年 2 月 15 日。在這個小節裡，我們會編寫一個實用的工具來做日期運算。我們會將日期轉成整數，起始點設置在 2000 年 1 月 1 日。我們會使用內建的 ``+`` 與 ``-`` 函數來處理這些數字，而當我們轉換完畢時，再將結果轉回日期。

要將日期轉成數字，我們需要從日期的單位中，算出總天數有多少。舉例來說，2004 年 11 月 13 日的天數總和，是從起始點至 2004 年有多少天，加上從 2004 年到 2004 年 11 月有多少天，再加上 13 天。

有一個我們會需要的東西是，一張列出非潤年每月份有多少天的表格。我們可以使用 Lisp 來推敲出這個表格的內容。我們從列出每月份的長度開始：

::

  > (setf mon '(31 28 31 30 31 30 31 31 30 31 30 31))
  (31 28 31 30 31 30 31 31 30 31 30 31)

我們可以通過應用 ``+`` 函數至這個列表來測試總長度：

::

  > (apply #'+ mon)
  365

現在如果我們反轉這個列表並使用 ``maplist`` 來應用 ``+`` 函數至每下一個 ``cdr`` 上，我們可以獲得從每個月份開始所累積的天數：

::

  > (setf nom (reverse mon))
  (31 30 31 30 31 31 30 31 30 31 28 31)
  > (setf sums (maplist #'(lambda (x)
                            (apply #'+ x))
                        nom))
  (365 334 304 273 243 212 181 151 120 90 59 31)

這些數字體現了從二月一號開始已經過了 31 天，從三月一號開始已經過了 59 天……等等。

我們剛剛建立的這個列表，可以轉換成一個向量，見圖 5.1，轉換日期至整數的程式。

::

  (defconstant month
    #(0 31 59 90 120 151 181 212 243 273 304 334 365))

  (defconstant yzero 2000)

  (defun leap? (y)
    (and (zerop (mod y 4))
         (or (zerop (mod y 400))
             (not (zerop (mod y 100))))))

  (defun date->num (d m y)
    (+ (- d 1) (month-num m y) (year-num y)))

  (defun month-num (m y)
    (+ (svref month (- m 1))
       (if (and (> m 2) (leap? y)) 1 0)))

  (defun year-num (y)
    (let ((d 0))
      (if (>= y yzero)
          (dotimes (i (- y yzero) d)
            (incf d (year-days (+ yzero i))))
          (dotimes (i (- yzero y) (- d))
            (incf d (year-days (+ y i)))))))

  (defun year-days (y) (if (leap? y) 366 365))

**圖 5.1 日期運算：轉換日期至數字**

典型 Lisp 程式的生命週期有四個階段：先寫好，然後讀入，接著編譯，最後執行。有件 Lisp 非常獨特的事情之一是，在這四個階段時， Lisp 一直都在那裡。可以在你的程式編譯 (參見 10.2 小節)或讀入時 (參見 14.3 小節) 來呼叫 Lisp。我們推導出 ``month`` 的過程示範了，如何在撰寫一個程式時使用 Lisp。

效率通常只跟第四個階段有關係，運行期（run-time）。在前三個階段，你可以隨意的使用列表擁有的威力與靈活性，而不需要擔心效率。

若你使用圖 5.1 的程式來造一個時光機器（time machine），當你抵達時，人們大概會不同意你的日期。即使是相對近的現在，歐洲的日期也曾有過偏移，因爲人們會獲得更精準的每年有多長的概念。在說英語的國家，最後一次的不連續性出現在 1752 年，日期從 9 月 2 日跳到 9 月 14 日。

每年有幾天取決於該年是否是潤年。如果該年可以被四整除，我們說該年是潤年，除非該年可以被 100 整除，則該年非潤年 ── 而要是它可以被 400 整除，則又是潤年。所以 1904 年是潤年，1900 年不是，而 1600 年是。

要決定某個數是否可以被另個數整除，我們使用函數 ``mod`` ，返回相除後的餘數：

::

  > (mod 23 5)
  3
  > (mod 25 5)
  0

如果第一個實參除以第二個實參的餘數爲 0，則第一個實參是可以被第二個實參整除的。函數 ``leap?`` 使用了這個方法，來決定它的實參是否是一個潤年：

::

  > (mapcar #'leap? '(1904 1900 1600))
  (T NIL T)

我們用來轉換日期至整數的函數是 ``date->num`` 。它返回日期中每個單位的天數總和。要找到從某月份開始的天數和，我們呼叫 ``month-num`` ，它在 ``month`` 中查詢天數，如果是在潤年的二月之後，則加一。

要找到從某年開始的天數和， ``date->num`` 呼叫 ``year-num`` ，它返回某年一月一日相對於起始點（2000.01.01）所代表的天數。這個函數的工作方式是從傳入的實參 ``y`` 年開始，朝著起始年（2000）往上或往下數。

::

  (defun num->date (n)
    (multiple-value-bind (y left) (num-year n)
      (multiple-value-bind (m d) (num-month left y)
        (values d m y))))

  (defun num-year (n)
    (if (< n 0)
        (do* ((y (- yzero 1) (- y 1))
              (d (- (year-days y)) (- d (year-days y))))
             ((<= d n) (values y (- n d))))
        (do* ((y yzero (+ y 1))
              (prev 0 d)
              (d (year-days y) (+ d (year-days y))))
             ((> d n) (values y (- n prev))))))

  (defun num-month (n y)
    (if (leap? y)
        (cond ((= n 59) (values 2 29))
              ((> n 59) (nmon (- n 1)))
              (t        (nmon n)))
        (nmon n)))

  (defun nmon (n)
    (let ((m (position n month :test #'<)))
      (values m (+ 1 (- n (svref month (- m 1)))))))

  (defun date+ (d m y n)
    (num->date (+ (date->num d m y) n)))

**圖 5.2 日期運算：轉換數字至日期**

圖 5.2 展示了程式的下半部份。函數 ``num->date`` 將整數轉換回日期。它呼叫了 ``num-year`` 函數，以日期的格式返回年，以及剩餘的天數。再將剩餘的天數傳給 ``num-month`` ，分解出月與日。

和 ``year-num`` 相同， ``num-year`` 從起始年往上或下數，一次數一年。並持續累積天數，直到它獲得一個絕對值大於或等於 ``n``  的數。如果它往下數，那麼它可以返回當前迭代中的數值。不然它會超過年份，然後必須返回前次迭代的數值。這也是爲什麼要使用 ``prev`` ， ``prev`` 在每次迭代時會存入 ``days`` 前次迭代的數值。

函數 ``num-month`` 以及它的子程式（subroutine） ``nmon`` 的行爲像是相反地 ``month-num`` 。他們從常數向量 ``month`` 的數值到位置，然而 ``month-num`` 從位置到數值。

圖 5.2 的前兩個函數可以合而爲一。與其返回數值給另一個函數， ``num-year`` 可以直接呼叫 ``num-month`` 。現在分成兩部分的程式，比較容易做交互測試，但是現在它可以工作了，下一步或許是把它合而爲一。

有了 ``date->num`` 與 ``num->date`` ，日期運算是很簡單的。我們在 ``date+`` 裡使用它們，可以從特定的日期做加減。如果我們想透過 ``date+`` 來知道 1997 年 12 月 17 日六十天之後的日期:

::

  > (multiple-value-list (date+ 17 12 1997 60))
  (15 2 1998)

我們得到，1998 年 2 月 15 日。

Chapter 5 總結 (Summary)
============================

1. Common Lisp 有三個基本的區塊建構子： ``progn`` ；允許返回的 ``block`` ；以及允許 ``goto`` 的 ``tagbody`` 。很多內建的運算子隱含在區塊裡。

2. 進入一個新的詞法語境，概念上等同於函數呼叫。

3. Common Lisp 提供了適合不同情況的條件式。每個都可以使用 ``if`` 來定義。

4. 有數個相似迭代運算子的變種。

5. 表達式可以返回多個數值。

6. 計算過程可以被中斷以及保護，保護可使其免於中斷所造成的後果。

Chapter 5 練習 (Exercises)
==================================

1. 將下列表達式翻譯成沒有使用 ``let`` 與 ``let*`` ，並使同樣的表達式不被求值 2 次。

::

  (a) (let ((x (car y)))
        (cons x x))
  (b) (let* ((w (car x))
             (y (+ w z)))
        (cons w y))

2. 使用 ``cond`` 重寫 29 頁的 ``mystery`` 函數。（譯註: 第二章的練習第 5 題的 (b) 部分)

3. 定義一個返回其實參平方的函數，而當實參是一個正整數且小於等於 5 時，不要計算其平方。

4. 使用 ``case`` 與 ``svref`` 重寫 ``month-num`` (圖 5.1)。

5. 定義一個迭代與遞迴版本的函數，接受一個物件 x 與向量 v ，並返回一個列表，包含了向量 v 當中，所有直接在 ``x`` 之前的物件：

::

  > (precedes #\a "abracadabra")
  (#\c #\d #\r)

6. 定義一個迭代與遞迴版本的函數，接受一個物件與列表，並返回一個新的列表，在原本列表的物件之間加上傳入的物件：

::

  > (intersperse '- '(a b c d))
  (A - B - C - D)

7. 定義一個接受一系列數字的函數，並在若且唯若每一對（pair）數字的差爲一時，返回真，使用

::

  (a) 遞迴
  (b) do
  (c) mapc 與 return

8. 定義一個單遞迴函數，返回兩個值，分別是向量的最大與最小值。

9. 圖 3.12 的程式在找到一個完整的路徑時，仍持續遍歷佇列。在搜索範圍大時，這可能會產生問題。

::

  (a) 使用 catch 與 throw 來變更程式，使其找到第一個完整路徑時，直接返回它。
  (b) 重寫一個做到同樣事情的程式，但不使用 catch 與 throw。
