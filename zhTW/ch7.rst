.. highlight:: cl

第七章：輸入與輸出
***************************************************

Common Lisp 有著威力強大的 I/O 工具。針對輸入以及一些普遍讀取字元的函數，我們有 ``read`` ，包含了一個完整的解析器 (parser)。針對輸出以及一些普遍寫出字元的函數，我們有 ``format`` ，它自己幾乎就是一個語言。本章介紹了所有基本的概念。

Common Lisp 有兩種流 (streams)，字元流與二進制流。本章描述了字元流的操作；二進制流的操作涵蓋在 14.2 節。

7.1 流 (Streams)
==================================

流是用來表示字元來源或終點的 Lisp 物件。要從檔案讀取或寫入，你將檔案作爲流打開。但流與檔案是不一樣的。當你在頂層讀入或印出時，你也可以使用流。你甚至可以創建可以讀取或寫入字串的流。

輸入預設是從 ``*standard-input*`` 流讀取。輸出預設是在 ``*standard-output*`` 流。最初它們大概會在相同的地方：一個表示頂層的流。

我們已經看過 ``read`` 與 ``format`` 是如何在頂層讀取與印出。前者接受一個應是流的選擇性參數，預設是 ``*standard-input*`` 。 ``format`` 的第一個參數也可以是一個流，但當它是 ``t`` 時，輸出被送到 ``*standard-output*`` 。所以我們目前爲止都只用到預設的流而已。我們可以在任何流上面做同樣的 I/O 操作。

路徑名（pathname）是一種指定一個檔案的可移植方式。路徑名包含了六個部分：host、device、directory、name、type 及 version。你可以通過呼叫 ``make-pathname`` 搭配一個或多個對應的關鍵字參數來產生一個路徑。在最簡單的情況下，你可以只指明名字，讓其他的部分留爲預設：

::

	> (setf path (make-pathname :name "myfile"))
	#P"myfile"

開啓一個檔案的基本函數是 ``open`` 。它接受一個路徑名 [1]_ 以及大量的選擇性關鍵字參數，而若是開啓成功時，返回一個指向檔案的流。

你可以在創建流時，指定你想要怎麼使用它。 無論你是要寫入流、從流讀取或是兩者皆是， ``direction`` 參數都會偵測到。三個對應的數值是 ``:input`` , ``:output`` , ``:io`` 。如果是用來輸出的流， ``if-exists`` 參數說明了如果檔案已經存在時該怎麼做；通常它應該是 ``:supercede`` (譯註: 取代)。所以要創建一個可以寫至 ``"myfile"`` 檔案的流，你可以：

::

	> (setf str (open path :direction :output
	                       :if-exists :supersede))
	#<Stream C017E6>

流的列印表示法因實現而異 (implementation-dependent)。

現在我們可以把這個流作爲第一個參數傳給 ``format`` ，它會在流印出，而不是頂層：

::

	> (format str "Something~%")
	NIL

如果我們在此時檢視這個檔案，輸出也許會、也許不會在那裡。某些實現會將輸出儲存成一塊 (chunks)再寫出。它也許不會出現，直到我們將流關閉：

::

	> (close str)
	NIL

當你使用完時，永遠記得關閉檔案；在你還沒關閉之前，內容是不保證會出現的。現在如果我們檢視檔案 "myfile" ，應該有一行：

::

	Something

如果我們只想從一個檔案讀取，我們可以開啓一個具有 ``:direction :input`` 的流 ：

::

	> (setf str (open path :direction :input))
	#<Stream C01C86>

我們可以對一個檔案使用任何輸入函數。7.2 節會更詳細的描述輸入。這裡作爲一個範例，我們將使用 ``read-line`` 從檔案來讀取一行文字：

::

	> (read-line str)
	"Something"
	> (close str)
	NIL

當你讀取完畢時，記得關閉檔案。

大部分時間我們不使用 ``open`` 與 ``close`` 來操作檔案的 I/O 。 ``with-open-file`` 宏通常更方便。它的第一個參數應該是一個列表，包含了變數名、伴隨著你想傳給 ``open`` 的參數。在這之後，它接受一個程式碼主體，它會被綁定至流的變數一起被求值，其中流是通過將剩餘的參數傳給 ``open`` 來創建的。之後這個流會被自動關閉。所以整個檔案寫入動作可以表示爲：

::

  (with-open-file (str path :direction :output
                            :if-exists :supercede)
    (format str "Something~%"))

``with-open-file`` 宏將 ``close`` 放在 ``unwind-protect`` 裡 (參見 92 頁，譯註: 5.6 節)，即使一個錯誤打斷了主體的求值，檔案是保證會被關閉的。

7.2 輸入 (Input)
===============================

兩個最受歡迎的輸入函數是 ``read-line`` 及 ``read`` 。前者讀入換行符 (newline)之前的所有字元，並用字串返回它們。它接受一個選擇性流參數 (optional stream argument)；若流忽略時，預設爲 ``*standard-input*`` :

::

	> (progn
	    (format t "Please enter your name: ")
	    (read-line))
	Please enter your name: Rodrigo de Bivar
	"Rodrigo de Bivar"
	NIL

譯註：Rodrigo de Bivar 人稱熙德 (El cid)，十一世紀的西班牙民族英雄。

如果你想要原封不動的輸出，這是你該用的函數。(第二個返回值只在 ``read-line`` 在遇到換行符之前，用盡輸入時返回真。)

在一般情況下， ``read-line`` 接受四個選擇性參數: 一個流；一個參數用來決定遇到 ``end-of-file`` 時，是否產生錯誤；若前一個參數爲 ``nil`` 時，該返回什麼；第四個參數 (在 235 頁討論)通常可以省略。

所以要在頂層顯示一個檔案的內容，我們可以使用下面這個函數：

::

	(defun pseudo-cat (file)
	  (with-open-file (str file :direction :input)
	    (do ((line (read-line str nil 'eof)
	               (read-line str nil 'eof)))
	        ((eql line 'eof))
	      (format t "~A~%" line))))

如果我們想要把輸入解析爲 Lisp 物件，使用 ``read`` 。這個函數恰好讀取一個表達式，在表達式結束時停止讀取。所以可以讀取多於或少於一行。而當然它所讀取的內容必須是合法的 Lisp 語法。

如果我們在頂層使用 ``read`` ，它會讓我們在表達式裡面，想用幾個換行符就用幾個：

::

	> (read)
	(a
	b
	c)
	(A B C)

換句話說，如果我們在一行裡面輸入許多表達式， ``read`` 會在第一個表達式之後，停止處理字元，留下剩餘的字元給之後讀取這個流的函數處理。所以如果我們在一行輸入多個表達式，來回應 ``ask-number`` (20 頁。譯註：2.10 小節)所印出提示符，會發生如下情形:

::

	> (ask-number)
	Please enter a number. a b
	Please enter a number. Please enter a number. 43
	43

兩個連續的提示符 (successive prompts)在第二行被印出。第一個 ``read`` 呼叫會返回 ``a`` ，而它不是一個數字，所以函數再次要求一個數字。但第一個 ``read``	只讀取到 ``a`` 的結尾。所以下一個 ``read`` 呼叫返回 ``b`` ，導致了下一個提示符。

你或許想要避免使用 ``read`` 來直接處理使用者的輸入。前述的函數若使用 ``read-line`` 來獲得使用者輸入會比較好，然後對結果字串呼叫 ``read-from-string`` 。這個函數接受一個字串，並返回第一個讀取的表達式:

::

	> (read-from-string "a b c")
	A
	2

它同時返回第二個值，一個指出停止讀取字串時的位置的數字。

在一般情況下， ``read-from-string`` 可以接受兩個選擇性參數與三個關鍵字參數。兩個選擇性參數是 ``read`` 的第三、第四個參數: 一個 end-of-file (這個情況是字串) 決定是否報錯，若不報錯該返回什麼。關鍵字參數 ``:start`` 及 ``:end`` 可以用來劃分從字串的哪裡開始讀。

所有的這些輸入函數是由基本函數 (primitive) ``read-char`` 所定義的，它讀取一個字元。它接受四個與 ``read`` 及 ``read-line`` 一樣的選擇性參數。Common Lisp 也定義一個函數叫做 ``peek-char`` ，跟 ``read-char`` 類似，但不會將字元從流中移除。

7.3 輸出 (Output)
================================

三個最簡單的輸出函數是 ``prin1`` , ``princ`` 以及 ``terpri`` 。這三個函數的最後一個參數皆爲選擇性的流參數，預設是 ``*standard-output*`` 。

``prin1`` 與 ``princ`` 的差別大致在於 ``prin1`` 給程式產生輸出，而 ``princ`` 給人類產生輸出。所以舉例來說， ``prin1`` 會印出字串左右的雙引號，而 ``princ`` 不會:

::

	> (prin1 "Hello")
	"Hello"
	"Hello"
	> (princ "Hello")
	Hello
	"Hello"

兩者皆返回它們的第一個參數 (譯註: 第二個值是返回值) ── 順道一提，是用 ``prin1`` 印出。 ``terpri`` 僅印出一新行。

有這些函數的背景知識在解釋更爲通用的 ``format`` 是很有用的。這個函數幾乎可以用在所有的輸出。他接受一個流 (或 ``t`` 或 ``nil`` )、一個格式化字串 (format string)以及零個或多個額外的參數。格式化字串可以包含特定的格式化指令 (format directives)，這些指令前面有波浪號 ``~`` 。某些格式化指令作爲字串的佔位符 (placeholder)使用。這些位置會被格式化字串之後，所給入參數的表示法所取代。

如果我們把 ``t`` 作爲第一個參數，輸出會被送至 ``*standard-output*`` 。如果我們給 ``nil`` ， ``format`` 會返回一個它會如何印出的字串。爲了保持簡短，我們會在所有的範例裡示範怎麼做。

由於每人的觀點不同， ``format`` 可以是令人驚訝的強大或是極爲可怕的複雜。有大量的格式化指令可用，而只有少部分會被大多數程式設計師使用。兩個最常用的格式化指令是 ``~A`` 以及 ``~%`` 。(你使用 ``~a`` 或 ``~A`` 都沒關係，但後者較常見，因爲它讓格式化指令看起來一目了然。) 一個 ``~A`` 是一個值的佔位符，它會像是用 ``princ`` 印出一般。一個 ``~%`` 代表著一個換行符 (newline)。

::

	> (format nil "Dear ~A, ~% Our records indicate..."
						"Mr. Malatesta")
	"Dear Mr. Malatesta,
	   Our records indicate..."

這裡 ``format`` 返回了一個值，由一個含有換行符的字串組成。

``~S`` 格式化指令像是 ``~A`` ，但它使用 ``prin1`` 印出物件，而不是 ``princ`` 印出:

::

	> (format t "~S  ~A" "z" "z")
	"z" z
	NIL

格式化指令可以接受參數。 ``~F`` 用來印出向右對齊 (right-justified)的浮點數，可接受五個參數:

1. 要印出字元的總數。預設是數字的長度。

2. 小數之後要印幾位數。預設是全部。

3. 小數點要往右移幾位 (即等同於將數字乘 10)。預設是沒有。

4. 若數字太長無法滿足第一個參數時，所要印出的字元。如果沒有指定字元，一個過長的數字會儘可能使用它所需的空間被印出。

5. 數字開始印之前左邊的字元。預設是空白。

下面是一個有五個參數的罕見例子:

::

	? (format nil "~10,2,0,'*,' F" 26.21875)
	"     26.22"

這是原本的數字取至小數點第二位、(小數點向左移 0 位)、在 10 個字元的空間裡向右對齊，左邊補滿空白。注意作爲參數給入是寫成 ``'*`` 而不是 ``#\*`` 。由於數字塞得下 10 個字元，不需要使用第四個參數。

所有的這些參數都是選擇性的。要使用預設值你可以直接忽略對應的參數。如果我們想要做的是，印出一個小數點取至第二位的數字，我們可以說:

::

	> (format nil "~,2,,,F" 26.21875)
	"26.22"

你也可以忽略一系列的尾隨逗號 (trailing commas)，前面指令更常見的寫法會是:

::

	> (format nil "~,2F" 26.21875)
	"26.22"

**警告:** 當 ``format`` 取整數時，它不保證會向上進位或向下舍入。就是說 ``(format nil "~,1F" 1.25)`` 可能會是 ``"1.2"`` 或 ``"1.3"`` 。所以如果你使用 ``format`` 來顯示資訊時，而使用者期望看到某種特定取整數方式的數字 (如: 金額數量)，你應該在印出之前先顯式地取好整數。

7.4 範例：字串代換 (Example: String Substitution)
==============================================================

作爲一個 I/O 的範例，本節示範如何寫一個簡單的程式來對文字檔案做字串替換。我們即將寫一個可以將一個檔案中，舊的字串 ``old`` 換成某個新的字串 ``new`` 的函數。最簡單的實現方式是將輸入檔案裡的每一個字元與 ``old`` 的第一個字元比較。如果沒有匹配，我們可以直接印出該字元至輸出。如果匹配了，我們可以將輸入的下一個字元與 ``old`` 的第二個字元比較，等等。如果輸入字元與 ``old`` 完全相等時，我們有一個成功的匹配，則我們印出 ``new`` 至檔案。

而要是 ``old`` 在匹配途中失敗了，會發生什麼事呢？舉例來說，假設我們要找的模式 (pattern)是 ``"abac"`` ，而輸入檔案包含的是 ``"ababac"`` 。輸入會一直到第四個字元才發現不匹配，也就是在模式中的 ``c`` 以及輸入的 ``b`` 才發現。在此時我們可以將原本的 ``a`` 寫至輸出檔案，因爲我們已經知道這裡沒有匹配。但有些我們從輸入讀入的字元還是需要留著: 舉例來說，第三個 ``a`` ，確實是成功匹配的開始。所以在我們要實現這個算法之前，我們需要一個地方來儲存，我們已經從輸入讀入的字元，但之後仍然需要的字元。

一個暫時儲存輸入的佇列 (queue)稱作緩衝區 (buffer)。在這個情況裡，因爲我們知道我們不需要儲存超過一個預定的字元量，我們可以使用一個叫做環狀緩衝區 ``ring buffer`` 的資料結構。一個環狀緩衝區實際上是一個向量。是使用的方式使其成爲環狀: 我們將之後的元素所輸入進來的值儲存起來，而當我們到達向量結尾時，我們重頭開始。如果我們不需要儲存超過 ``n`` 個值，則我們只需要一個長度爲 ``n`` 或是大於 ``n`` 的向量，這樣我們就不需要覆寫正在用的值。

在圖 7.1 的程式裡，實現了環狀緩衝區的操作。 ``buf`` 有五個欄位 (field): 一個包含存入緩衝區的向量，四個其它欄位用來放指向向量的索引 (indices)。兩個索引是 ``start`` 與 ``end`` ，任何環狀緩衝區的使用都會需要這兩個索引: ``start`` 指向緩衝區的第一個值，當我們取出一個值時， ``start`` 會遞增 (incremented)； ``end`` 指向緩衝區的最後一個值，當我們插入一個新值時， ``end`` 會遞增。

另外兩個索引， ``used`` 以及 ``new`` ，是我們需要給這個應用的基本環狀緩衝區所加入的東西。它們會介於 ``start`` 與 ``end`` 之間。實際上，它總是符合

::

	start ≤ used ≤ new ≤ end

你可以把 ``used`` 與 ``new`` 想成是當前匹配 (current match) 的 ``start`` 與 ``end`` 。當我們開始一輪匹配時， ``used`` 會等於 ``start`` 而 ``new`` 會等於 ``end`` 。當下一個字元 (successive character)匹配時，我們需要遞增 ``used`` 。當 ``used`` 與 ``new`` 相等時，我們將開始匹配時，所有存在緩衝區的字元讀入。我們不想要使用超過從匹配時所存在緩衝區的字元，或是重複使用同樣的字元。因此這個 ``new`` 索引，開始等於 ``end`` ，但它不會在一輪匹配我們插入新字元至緩衝區一起遞增。

函數 ``bref`` 接受一個緩衝區與一個索引，並返回索引所在位置的元素。藉由使用 ``index`` 對向量的長度取 ``mod`` ，我們可以假裝我們有一個任意長的緩衝區。呼叫 ``(new-buf n)`` 會產生一個新的緩衝區，能夠容納 ``n`` 個物件。

要插入一個新值至緩衝區，我們將使用 ``buf-insert`` 。它將 ``end`` 遞增，並把新的值放在那個位置 (譯註: 遞增完的位置)。相反的 ``buf-pop`` 返回一個緩衝區的第一個數值，接著將 ``start`` 遞增。任何環狀緩衝區都會有這兩個函數。

::

	(defstruct buf
	  vec (start -1) (used -1) (new -1) (end -1))

	(defun bref (buf n)
	  (svref (buf-vec buf)
	         (mod n (length (buf-vec buf)))))

	(defun (setf bref) (val buf n)
	  (setf (svref (buf-vec buf)
	               (mod n (length (buf-vec buf))))
	        val))

	(defun new-buf (len)
	  (make-buf :vec (make-array len)))

	(defun buf-insert (x b)
	  (setf (bref b (incf (buf-end b))) x))

	(defun buf-pop (b)
	  (prog1
	    (bref b (incf (buf-start b)))
	    (setf (buf-used b) (buf-start b)
	          (buf-new  b) (buf-end   b))))

	(defun buf-next (b)
	  (when (< (buf-used b) (buf-new b))
	    (bref b (incf (buf-used b)))))

	(defun buf-reset (b)
	  (setf (buf-used b) (buf-start b)
	        (buf-new  b) (buf-end   b)))

	(defun buf-clear (b)
	  (setf (buf-start b) -1 (buf-used  b) -1
	        (buf-new   b) -1 (buf-end   b) -1))

	(defun buf-flush (b str)
	  (do ((i (1+ (buf-used b)) (1+ i)))
	      ((> i (buf-end b)))
	    (princ (bref b i) str)))

**圖 7.1 環狀緩衝區的操作**

接下來我們需要兩個特別爲這個應用所寫的函數: ``buf-next`` 從緩衝區讀取一個值而不取出，而 ``buf-reset`` 重置 ``used`` 與 ``new`` 到初始值，分別是 ``start`` 與 ``end`` 。如果我們已經把至 ``new`` 的值全部讀取完畢時， ``buf-next`` 返回 ``nil`` 。區別這個值與實際的值不會產生問題，因爲我們只把值存在緩衝區。

最後 ``buf-flush`` 透過將所有作用的元素，寫至由第二個參數所給入的流，而 ``buf-clear`` 通過重置所有的索引至 ``-1`` 將緩衝區清空。

在圖 7.1 定義的函數被圖 7.2 所使用，包含了字串替換的程式。函數 ``file-subst`` 接受四個參數；一個查詢字串，一個替換字串，一個輸入檔案以及一個輸出檔案。它創建了代表每個檔案的流，然後呼叫 ``stream-subst`` 來完成實際的工作。

第二個函數 ``stream-subst`` 使用本節開始所勾勒的算法。它一次從輸入流讀一個字元。直到輸入字元匹配要尋找的字串時，直接寫至輸出流 (1)。當一個匹配開始時，有關字元在緩衝區 ``buf`` 排隊等候 (2)。

變數 ``pos`` 指向我們想要匹配的字元在尋找字串的所在位置。如果 ``pos`` 等於這個字串的長度，我們有一個完整的匹配，則我們將替換字串寫至輸出流，並清空緩衝區 (3)。如果在這之前匹配失敗，我們可以將緩衝區的第一個元素取出，並寫至輸出流，之後我們重置緩衝區，並從 ``pos`` 等於 0 重新開始 (4)。

::

	(defun file-subst (old new file1 file2)
	  (with-open-file (in file1 :direction :input)
	     (with-open-file (out file2 :direction :output
	                                :if-exists :supercede)
	       (stream-subst old new in out))))

	(defun stream-subst (old new in out)
	  (let* ((pos 0)
	         (len (length old))
	         (buf (new-buf len))
	         (from-buf nil))
	    (do ((c (read-char in nil :eof)
	            (or (setf from-buf (buf-next buf))
	                (read-char in nil :eof))))
	        ((eql c :eof))
	      (cond ((char= c (char old pos))
	             (incf pos)
	             (cond ((= pos len)            ; 3
	                    (princ new out)
	                    (setf pos 0)
	                    (buf-clear buf))
	                   ((not from-buf)         ; 2
	                    (buf-insert c buf))))
	            ((zerop pos)                   ; 1
	             (princ c out)
	             (when from-buf
	               (buf-pop buf)
	               (buf-reset buf)))
	            (t                             ; 4
	             (unless from-buf
	               (buf-insert c buf))
	             (princ (buf-pop buf) out)
	             (buf-reset buf)
	             (setf pos 0))))
	    (buf-flush buf out)))

**圖 7.2 字串替換**

下列表格展示了當我們將檔案中的 ``"baro"`` 替換成 ``"baric"`` 所發生的事，其中檔案只有一個單字 ``"barbarous"`` :

+-----------+----------+-------+------+--------+------------+
| CHARACTER |  SOURCE  | MATCH | CASE | OUTPUT |   BUFFER   |
+===========+==========+=======+======+========+============+
| b         | file     |   b   |  2   |        | b          |
+-----------+----------+-------+------+--------+------------+
| a         | file     |   a   |  2   |        | b a        |
+-----------+----------+-------+------+--------+------------+
| r         | file     |   r   |  2   |        | b a r      |
+-----------+----------+-------+------+--------+------------+
| b         | file     |   o   |  4   | b      | b.a r b.   |
+-----------+----------+-------+------+--------+------------+
| a         | buffer   |   b   |  1   | a      | a.r b.     |
+-----------+----------+-------+------+--------+------------+
| r         | buffer   |   b   |  1   | r      | r.b.       |
+-----------+----------+-------+------+--------+------------+
| b         | buffer   |   b   |  1   |        | r b:       |
+-----------+----------+-------+------+--------+------------+
| a         | file     |   a   |  2   |        | r b:a      |
+-----------+----------+-------+------+--------+------------+
| r         | file     |   r   |  2   |        | r b:a      |
+-----------+----------+-------+------+--------+------------+
| o         | file     |   o   |  3   | baric  | r b:a r    |
+-----------+----------+-------+------+--------+------------+
| u         | file     |   b   |  1   | u      |            |
+-----------+----------+-------+------+--------+------------+
| a         | file     |   b   |  1   | s      |            |
+-----------+----------+-------+------+--------+------------+

第一欄是當前字元 ── ``c`` 的值；第二欄顯示是從緩衝區或是直接從輸入流讀取；第三欄顯示需要匹配的字元 ── ``old`` 的第 **posth** 字元；第四欄顯示那一個條件式 (case)被求值作爲結果；第五欄顯示被寫至輸出流的字元；而最後一欄顯示緩衝區之後的內容。在最後一欄裡， ``used`` 與 ``new`` 的位置一樣，由一個冒號 ( ``:`` colon)表示。

在檔案 ``"test1"`` 裡有如下文字：

::

	The struggle between Liberty and Authority is the most conspicuous feature
	in the portions of history with which we are earliest familiar, particularly
	in that of Greece, Rome, and England.

在我們對 ``(file-subst " th" " z" "test1" "test2")`` 求值之後，讀取檔案 ``"test2"`` 爲:

::

	The struggle between Liberty and Authority is ze most conspicuous feature
	in ze portions of history with which we are earliest familiar, particularly
	in zat of Greece, Rome, and England.

爲了使這個例子儘可能的簡單，圖 7.2 的程式只將一個字串換成另一個字串。很容易擴展爲搜索一個模式而不是一個字面字串。你只需要做的是，將 ``char=`` 呼叫換成一個你想要的更通用的匹配函數呼叫。

7.5 宏字元 (Macro Characters)
=======================================

一個宏字元 (macro character)是獲得 ``read`` 特別待遇的字元。比如小寫的 ``a`` ，通常與小寫 ``b`` 一樣處理，但一個左括號就不同了: 它告訴 Lisp 開始讀入一個列表。

一個宏字元或宏字元組合也稱作 ``read-macro`` (讀取宏) 。許多 Common Lisp 預定義的讀取宏是縮寫。比如說引用 (Quote): 讀入一個像是 ``'a`` 的表達式時，它被讀取器展開成 ``(quote a)`` 。當你輸入引用的表達式 (quoted expression)至頂層時，它們在讀入之時就會被求值，所以一般來說你看不到這樣的轉換。你可以透過顯式呼叫 ``read`` 使其現形:

::

	> (car (read-from-string "'a"))
	QUOTE

引用對於讀取宏來說是不尋常的，因爲它用單一字元表示。有了一個有限的字元集，你可以在 Common Lisp 裡有許多單一字元的讀取宏，來表示一個或更多字元。

這樣的讀取宏叫做派發 (dispatching)讀取宏，而第一個字元叫做派發字元 (dispatching character)。所有預定義的派發讀取宏使用井號 ( ``#`` )作爲派發字元。我們已經見過好幾個。舉例來說， ``#'`` 是 ``(function ...)`` 的縮寫，同樣的 ``'`` 是 ``(quote ...)`` 的縮寫。

其它我們見過的派發讀取宏包括 ``#(...)`` ，產生一個向量； ``#nA(...)`` 產生陣列； ``#\`` 產生一個字元； ``#S(n ...)`` 產生一個結構。當這些型別的每個物件被 ``prin1`` 顯示時 (或是 ``format`` 搭配 ``~S``)，它們使用對應的讀取宏 [2]_ 。這表示著你可以寫出或讀回這樣的物件:

::

	> (let ((*print-array* t))
	    (vectorp (read-from-string (format nil "~S"
	                                       (vector 1 2)))))
	T

當然我們拿回來的不是同一個向量，而是具有同樣元素的新向量。

不是所有物件被顯示時都有著清楚 (distinct)、可讀的形式。舉例來說，函數與雜湊表，傾向於這樣 ``#<...>`` 被顯示。實際上 ``#<...>`` 也是一個讀取宏，但是特別用來產生當遇到 ``read`` 的錯誤。函數與雜湊表不能被寫出與讀回來，而這個讀取宏確保使用者不會有這樣的幻覺。 [3]_

當你定義你自己的事物表示法時 (舉例來說，結構的印出函數)，你要將此準則記住。要不使用一個可以被讀回來的表示法，或是使用 ``#<...>`` 。

Chapter 7 總結 (Summary)
============================

1. 流是輸入的來源或終點。在字元流裡，輸入輸出是由字元組成。

2. 預設的流指向頂層。新的流可以由開啓檔案產生。

3. 你可以解析物件、字元組成的字串、或是單獨的字元。

4. ``format`` 函數提供了完整的輸出控制。

5. 爲了要替換文字檔案中的字串，你需要將字元讀入緩衝區。

6. 當 ``read`` 遇到一個宏字元像是 ``'`` ，它呼叫相關的函數。

Chapter 7 練習 (Exercises)
==================================

1. 定義一個函數，接受一個檔案名並返回一個由字串組成的列表，來表示檔案裡的每一行。

2. 定義一個函數，接受一個檔案名並返回一個由表達式組成的列表，來表示檔案裡的每一行。

3. 假設有某種格式的檔案檔案，註解是由 ``%`` 字元表示。從這個字元開始直到行尾都會被忽略。定義一個函數，接受兩個檔案名稱，並拷貝第一個檔案的內容去掉註解，寫至第二個檔案。

4. 定義一個函數，接受一個二維浮點陣列，將其用簡潔的欄位顯示。每個元素應印至小數點二位，一欄十個字元寬。（假設所有的字元可以容納）。你會需要 ``array-dimensions`` (參見 361 頁，譯註: Appendix D)。

5. 修改 ``stream-subst`` 來允許萬用字元 (wildcard) 可以在模式中使用。若字元 ``+`` 出現在 ``old`` 裡，它應該匹配任何輸入字元。

6. 修改 ``stream-subst`` 來允許模式可以包含一個用來匹配任何數字的元素，以及一個可以匹配任何英文字元的元素或是一個可以匹配任何字元的元素。模式必須可以匹配任何特定的輸入字元。(提示: ``old`` 可以不是一個字串。)


.. rubric:: 腳註

.. [1] 你可以給一個字串取代路徑名，但這樣就不可攜了 (portable)。

.. [2] 要讓向量與陣列這樣被顯示，將 ``*print-array*`` 設爲真。

.. [3] Lisp 不能只用 ``#'`` 來表示函數，因爲 ``#'`` 本身無法提供表示閉包的方式。
