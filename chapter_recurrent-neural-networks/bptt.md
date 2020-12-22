# Zamanda Geri Yayılma
:label:`sec_bptt`

Şimdiye kadar defalarca gibi şeyler ima var
*patlayan degradeler*,
*kaybolan degradeler*,
ve ihtiyaç
*RNN'ler için degrade* ayırın.
Örneğin, :numref:`sec_rnn_scratch`'te dizideki `detach` işlevini çağırdık. Bunların hiçbiri gerçekten tam olarak açıklanmadı, hızlı bir model inşa edebilmek ve nasıl çalıştığını görmek için. Bu bölümde, sıralı modeller için geri yayılımın ayrıntılarını ve matematiğin neden (ve nasıl) çalıştığını biraz daha derinlemesine inceleyeceğiz.

RNN'leri ilk uyguladığımızda degrade patlamanın bazı etkileriyle karşılaştık (:numref:`sec_rnn_scratch`). Özellikle, egzersizleri çözdüyseniz, doğru yakınsamayı sağlamak için degrade kırpmanın hayati önem taşıdığını görürdünüz. Bu sorunun daha iyi anlaşılmasını sağlamak için, bu bölümde degradelerin sıra modelleri için nasıl hesaplandığını inceleyecektir. Nasıl çalıştığına dair kavramsal olarak yeni bir şey olmadığını unutmayın. Sonuçta, biz hala sadece degradeleri hesaplamak için zincir kuralını uyguluyoruz. Bununla birlikte, geri yayılımı (:numref:`sec_backprop`) tekrar incelerken buna değer.

:numref:`sec_backprop`'te MLP'lerde ileri ve geri yayılımlarını ve hesaplama grafiklerini tanımladık. Bir RNN'de ileriye yayılma nispeten basittir.
*Zamanla Backpropagation aslında belirli bir
RNN'lerde backpropagasyon uygulaması :cite:`Werbos.1990`. Model değişkenleri ve parametreleri arasındaki bağımlılıkları elde etmek için bir RNN'nin hesaplama grafiğini birer kerede bir adım genişletmemizi gerektirir. Ardından, zincir kuralına bağlı olarak, degradeleri hesaplamak ve depolamak için geri yayılım uygularız. Diziler oldukça uzun olabileceğinden, bağımlılık oldukça uzun olabilir. Örneğin, 1000 karakterlik bir dizi için, ilk belirteç nihai konumdaki belirteç üzerinde potansiyel olarak önemli bir etkiye sahip olabilir. Bu gerçekten hesaplamalı olarak mümkün değildir (çok uzun sürer ve çok fazla bellek gerektirir) ve biz bu çok zor gradyan ulaşmadan önce 1000'den fazla matris ürünü gerektirir. Bu, hesaplamalı ve istatistiksel belirsizliklerle dolu bir süreçtir. Aşağıda neler olduğunu ve bunu pratikte nasıl ele alacağımızı aydınlatacağız.

## RNN'lerde Gradyanların Analizi
:label:`subsec_bptt_analysis`

Bir RNN'nin nasıl çalıştığını anlatan basitleştirilmiş bir modelle başlıyoruz. Bu model, gizli durumun özellikleri ve nasıl güncellendiği hakkındaki ayrıntıları yok sayar. Buradaki matematiksel gösterim, skalaları, vektörleri ve matrisleri eskiden olduğu gibi açıkça ayırt etmez. Bu detaylar analiz için önemli değildir ve sadece bu alt bölümdeki notasyonu yığılmaya hizmet edecektir.

Bu basitleştirilmiş modelde, $h_t$'yi gizli durum olarak, $x_t$'yı giriş olarak ve $t$'deki çıkış olarak $o_t$'u gösteriyoruz. :numref:`subsec_rnn_w_hidden_states`'teki tartışmalarımızı hatırlayın, giriş ve gizli durum, gizli katmandaki bir ağırlık değişkeni ile çarpılacak şekilde birleştirilebilir. Böylece, sırasıyla gizli katmanın ve çıkış katmanının ağırlıklarını belirtmek için $w_h$ ve $w_o$'yi kullanırız. Sonuç olarak, her seferinde gizli durumlar ve çıkışlar aşağıdaki gibi açıklanabilir:

$$\begin{aligned}h_t &= f(x_t, h_{t-1}, w_h),\\o_t &= g(h_t, w_o),\end{aligned}$$
:eqlabel:`eq_bptt_ht_ot`

burada $f$ ve $g$, sırasıyla gizli katmanın ve çıkış katmanının dönüşümleridir. Bu nedenle, tekrarlayan hesaplama yoluyla birbirine bağlı $\{\ldots, (x_{t-1}, h_{t-1}, o_{t-1}), (x_{t}, h_{t}, o_t), \ldots\}$ değerler zincirine sahibiz. İleri yayılma oldukça basittir. İhtiyacımız olan tek şey $(x_t, h_t, o_t)$ üçlüleri bir seferde bir adım atmaktır. Çıktı $o_t$ ve istenen etiket $y_t$ arasındaki tutarsızlık daha sonra tüm $T$ zaman adımlarında objektif bir işlev tarafından değerlendirilir

$$L(x_1, \ldots, x_T, y_1, \ldots, y_T, w_h, w_o) = \frac{1}{T}\sum_{t=1}^T l(y_t, o_t).$$

Geri yayılma için, özellikle $L$ objektif fonksiyonun $w_h$ parametreleri ile ilgili olarak degradeleri hesapladığımızda konular biraz daha zordur. Belirli olmak gerekirse, zincir kuralına göre,

$$\begin{aligned}\frac{\partial L}{\partial w_h}  & = \frac{1}{T}\sum_{t=1}^T \frac{\partial l(y_t, o_t)}{\partial w_h}  \\& = \frac{1}{T}\sum_{t=1}^T \frac{\partial l(y_t, o_t)}{\partial o_t} \frac{\partial g(h_t, w_h)}{\partial h_t}  \frac{\partial h_t}{\partial w_h}.\end{aligned}$$
:eqlabel:`eq_bptt_partial_L_wh`

Ürünün :eqref:`eq_bptt_partial_L_wh`'teki birinci ve ikinci faktörlerinin hesaplanması kolaydır. $\partial h_t/\partial w_h$, $h_t$'da $w_h$ parametresinin etkisini tekrar hesaplamamız gerektiğinden, işlerin zor olduğu üçüncü faktör $\partial h_t/\partial w_h$. :eqref:`eq_bptt_ht_ot`'teki tekrarlayan hesaplamaya göre, $h_t$ $h_{t-1}$ ve $w_h$'ye bağlıdır, burada $h_{t-1}$'in hesaplanması da $w_h$'ye bağlıdır. Böylece, zincir kuralı verim kullanarak

$$\frac{\partial h_t}{\partial w_h}= \frac{\partial f(x_{t},h_{t-1},w_h)}{\partial w_h} +\frac{\partial f(x_{t},h_{t-1},w_h)}{\partial h_{t-1}} \frac{\partial h_{t-1}}{\partial w_h}.$$
:eqlabel:`eq_bptt_partial_ht_wh_recur`

Yukarıdaki degrade türetmek için, $t=1, 2,\ldots$ için $a_{0}=0$ ve $a_{t}=b_{t}+c_{t}a_{t-1}$ tatmin edici üç diziye sahip olduğumuzu varsayalım. Sonra $t\geq 1$ için göstermek kolaydır

$$a_{t}=b_{t}+\sum_{i=1}^{t-1}\left(\prod_{j=i+1}^{t}c_{j}\right)b_{i}.$$
:eqlabel:`eq_bptt_at`

$a_t$, $b_t$ ve $c_t$ yerine göre

$$\begin{aligned}a_t &= \frac{\partial h_t}{\partial w_h},\\
b_t &= \frac{\partial f(x_{t},h_{t-1},w_h)}{\partial w_h}, \\
c_t &= \frac{\partial f(x_{t},h_{t-1},w_h)}{\partial h_{t-1}},\end{aligned}$$

:eqref:`eq_bptt_partial_ht_wh_recur`'teki degrade hesaplama $a_{t}=b_{t}+c_{t}a_{t-1}$'yı karşılar. Böylece, :eqref:`eq_bptt_at` başına, :eqref:`eq_bptt_partial_ht_wh_recur`'teki tekrarlayan hesaplamayı kaldırabiliriz.

$$\frac{\partial h_t}{\partial w_h}=\frac{\partial f(x_{t},h_{t-1},w_h)}{\partial w_h}+\sum_{i=1}^{t-1}\left(\prod_{j=i+1}^{t} \frac{\partial f(x_{j},h_{j-1},w_h)}{\partial h_{j-1}} \right) \frac{\partial f(x_{i},h_{i-1},w_h)}{\partial w_h}.$$
:eqlabel:`eq_bptt_partial_ht_wh_gen`

$\partial h_t/\partial w_h$'i yinelemeli olarak hesaplamak için zincir kuralını kullanabilsek de, $t$ büyük olduğunda bu zincir çok uzun sürebilir. Bize bu sorunla başa çıkmak için stratejiler bir dizi tartışalım.

### Tam Hesaplama ###

Açıkçası, :eqref:`eq_bptt_partial_ht_wh_gen`'teki tam toplamı hesaplayabiliriz. Bununla birlikte, bu çok yavaştır ve degradeler patlayabilir, çünkü başlangıç koşullarındaki ince değişiklikler sonucu potansiyel olarak çok etkileyebilir. Yani, ilk koşullardaki minimum değişikliklerin sonuçta orantısız değişikliklere yol açtığı kelebek etkisine benzer şeyleri görebiliriz. Bu aslında tahmin etmek istediğimiz model açısından oldukça istenmeyen bir durumdur. Sonuçta, iyi genelleme sağlam tahminciler arıyoruz. Bu nedenle bu strateji pratikte neredeyse hiç kullanılmaz.

### Zaman Adımları###

Alternatif olarak, $\tau$ adımlardan sonra :eqref:`eq_bptt_partial_ht_wh_gen` toplamı kesebiliriz. Bu şimdiye kadar tartıştığımız şey, örneğin :numref:`sec_rnn_scratch`'teki degradeleri ayırdığımız zaman gibi. Bu, toplamı $\partial h_{t-\tau}/\partial w_h$'de sonlandırarak, gerçek degradenin *yaklaşıklığı* yol açar. Pratikte bu oldukça iyi çalışıyor. Genellikle zaman boyunca kesilmiş backpropgation olarak adlandırılır budur :cite:`Jaeger.2002`. Bunun sonuçlarından biri, modelin uzun vadeli sonuçlardan ziyade kısa vadeli etkilere odaklanması. Bu aslında *arzu edilir*, çünkü tahminleri daha basit ve daha kararlı modellere yöneltir.

### Rastgele kesme ###

Son olarak, biz yerine $\partial h_t/\partial w_h$ beklenti doğru ama dizisini keser rastgele bir değişken ile. Bu, önceden tanımlanmış $0 \leq \pi_t \leq 1$ ile $\xi_t$, burada $P(\xi_t = 0) = 1-\pi_t$ ve $P(\xi_t = \pi_t^{-1}) = \pi_t$, böylece $E[\xi_t] = 1$ bir dizi kullanılarak elde edilir. Bunu :eqref:`eq_bptt_partial_ht_wh_recur`'teki $\partial h_t/\partial w_h$'i değiştirmek için kullanıyoruz.

$$z_t= \frac{\partial f(x_{t},h_{t-1},w_h)}{\partial w_h} +\xi_t \frac{\partial f(x_{t},h_{t-1},w_h)}{\partial h_{t-1}} \frac{\partial h_{t-1}}{\partial w_h}.$$

Bu $\xi_t$ tanımından izler $E[z_t] = \partial h_t/\partial w_h$. Ne zaman $\xi_t = 0$ yinelenen hesaplama o zaman adım $t$ sona erer. Bu, uzun dizilerin nadir olduğu ancak uygun şekilde aşırı kilolu olduğu çeşitli uzunluklarda dizilerin ağırlıklı bir toplamına yol açar. Bu fikir Tallec ve Ollivier :cite:`Tallec.Ollivier.2017` tarafından önerildi.

### Stratejilerin Karşılaştırılması

![Comparing strategies for computing gradients in RNNs. From top to bottom: randomized truncation, regular truncation, and full computation.](../img/truncated-bptt.svg)
:label:`fig_truncated_bptt`

:numref:`fig_truncated_bptt`, RNN'ler için zaman içinde geri yayılım kullanan *The Time Machine* kitabının ilk birkaç karakterini analiz ederken üç stratejiyi göstermektedir:

* İlk satır, metni farklı uzunluklarda segmentlere ayıran randomize kesmedir.
* İkinci satır, metni aynı uzunlukta sonraya kıran düzenli kesimdir. Bu RNN deneylerinde yaptığımız şey.
* Üçüncü satır, hesaplama olarak mümkün olmayan bir ifadeye yol açan zaman içinde tam geri yayılımdır.

Ne yazık ki, teoride itiraz ederken, randomize kesme, büyük olasılıkla bir dizi faktöre bağlı olarak düzenli kesmeden çok daha iyi çalışmaz. Birincisi, geçmişe bir dizi geri yayılma adımından sonra bir gözlemin etkisi, pratikte bağımlılıkları yakalamak için oldukça yeterlidir. İkincisi, artan varyans, gradyanın daha fazla adımla daha doğru olduğu gerçeğine karşı koymaktadır. Üçüncüsü, aslında sadece kısa bir etkileşim aralığına sahip modeller istiyoruz. Bu nedenle, zaman içinde düzenli olarak kesilmiş geri yayılım, arzu edilebilecek hafif bir düzenleyici etkiye sahiptir.

## Ayrıntılı Zamanda Geri Yayılma

Genel prensibi tartıştıktan sonra, zaman içinde geriye yayılımı ayrıntılı olarak ele alalım. :numref:`subsec_bptt_analysis`'teki analizden farklı olarak, aşağıda, objektif fonksiyonun gradyanlarının tüm ayrıştırılmış model parametrelerine göre nasıl hesaplanacağını göstereceğiz. İşleri basit tutmak için, gizli katmandaki etkinleştirme işlevi kimlik eşlemelerini kullanan önyargı parametreleri olmayan bir RNN'yi göz önünde bulunduruyoruz ($\phi(x)=x$). Zaman adım $t$ için, tek örnek girdi ve etiket sırasıyla $\mathbf{x}_t \in \mathbb{R}^d$ ve $y_t$ olmasına izin verin. Gizli durum $\mathbf{h}_t \in \mathbb{R}^h$ ve çıkış $\mathbf{o}_t \in \mathbb{R}^q$ olarak hesaplanır

$$\begin{aligned}\mathbf{h}_t &= \mathbf{W}_{hx} \mathbf{x}_t + \mathbf{W}_{hh} \mathbf{h}_{t-1},\\
\mathbf{o}_t &= \mathbf{W}_{qh} \mathbf{h}_{t},\end{aligned}$$

burada $\mathbf{W}_{hx} \in \mathbb{R}^{h \times d}$, $\mathbf{W}_{hh} \in \mathbb{R}^{h \times h}$ ve $\mathbf{W}_{qh} \in \mathbb{R}^{q \times h}$ ağırlık parametreleridir. $l(\mathbf{o}_t, y_t)$ ile belirtin zaman adımındaki kaybı $t$. Bizim objektif fonksiyonu, üzerinde kayıp $T$ zaman adımlar dizinin başından bu nedenle

$$L = \frac{1}{T} \sum_{t=1}^T l(\mathbf{o}_t, y_t).$$

RNN'nin hesaplanması sırasında model değişkenleri ve parametreleri arasındaki bağımlılıkları görselleştirmek için, model için :numref:`fig_rnn_bptt`'te gösterildiği gibi bir hesaplama grafiği çizebiliriz. Örneğin, zaman adım 3, $\mathbf{h}_3$ gizli durumlarının hesaplanması model parametreleri $\mathbf{W}_{hx}$ ve $\mathbf{W}_{hh}$, son kez adım $\mathbf{h}_2$ gizli durumu ve geçerli zaman adım $\mathbf{x}_3$ giriş bağlıdır.

![Computational graph showing dependencies for an RNN model with three time steps. Boxes represent variables (not shaded) or parameters (shaded) and circles represent operators.](../img/rnn-bptt.svg)
:label:`fig_rnn_bptt`

Az önce belirtildiği gibi, :numref:`fig_rnn_bptt`'teki model parametreleri $\mathbf{W}_{hx}$, $\mathbf{W}_{hh}$ ve $\mathbf{W}_{qh}$'dır. Genel olarak, bu modelin eğitimi $\partial L/\partial \mathbf{W}_{hx}$, $\partial L/\partial \mathbf{W}_{hh}$ ve $\partial L/\partial \mathbf{W}_{qh}$ parametrelerine göre degrade hesaplama gerektirir. :numref:`fig_rnn_bptt`'teki bağımlılıklara göre, sırayla degradeleri hesaplamak ve depolamak için okların ters yönde geçebiliriz. Zincir kuralında farklı şekillerdeki matrislerin, vektörlerin ve skalaların çarpımını esnek bir şekilde ifade etmek için :numref:`sec_backprop`'te açıklandığı gibi $\text{prod}$ operatörünü kullanmaya devam ediyoruz.

Her şeyden önce, $t$ adımındaki herhangi bir zamanda model çıktısına göre objektif işlevi ayırt etmek oldukça basittir:

$$\frac{\partial L}{\partial \mathbf{o}_t} =  \frac{\partial l (\mathbf{o}_t, y_t)}{T \cdot \partial \mathbf{o}_t} \in \mathbb{R}^q.$$
:eqlabel:`eq_bptt_partial_L_ot`

Şimdi, objektif fonksiyonun gradyanını çıktı katmanındaki $\mathbf{W}_{qh}$ parametresine göre hesaplayabiliriz: $\partial L/\partial \mathbf{W}_{qh} \in \mathbb{R}^{q \times h}$. :numref:`fig_rnn_bptt` temel alınarak $L$ $\mathbf{W}_{qh}$'e $\mathbf{o}_1, \ldots, \mathbf{o}_T$ üzerinden $\mathbf{W}_{qh}$'e bağlıdır. Zincir kuralı verimleri kullanma

$$
\frac{\partial L}{\partial \mathbf{W}_{qh}}
= \sum_{t=1}^T \text{prod}\left(\frac{\partial L}{\partial \mathbf{o}_t}, \frac{\partial \mathbf{o}_t}{\partial \mathbf{W}_{qh}}\right)
= \sum_{t=1}^T \frac{\partial L}{\partial \mathbf{o}_t} \mathbf{h}_t^\top,
$$

burada $\partial L/\partial \mathbf{o}_t$ :eqref:`eq_bptt_partial_L_ot` tarafından verilir.

Daha sonra, :numref:`fig_rnn_bptt`'te gösterildiği gibi, $T$ son zaman adımında $L$ objektif işlev $\mathbf{h}_T$ yalnızca $\mathbf{o}_T$ üzerinden gizli duruma bağlıdır. Bu nedenle, zincir kuralını kullanarak $\partial L/\partial \mathbf{h}_T \in \mathbb{R}^h$'i kolayca bulabiliriz:

$$\frac{\partial L}{\partial \mathbf{h}_T} = \text{prod}\left(\frac{\partial L}{\partial \mathbf{o}_T}, \frac{\partial \mathbf{o}_T}{\partial \mathbf{h}_T} \right) = \mathbf{W}_{qh}^\top \frac{\partial L}{\partial \mathbf{o}_T}.$$
:eqlabel:`eq_bptt_partial_L_hT_final_step`

$L$ $\mathbf{h}_t$ üzerinden $\mathbf{h}_{t+1}$ ve $\mathbf{o}_t$ numaralı objektif fonksiyonun $\mathbf{h}_t$'ya bağlı olduğu herhangi bir zaman adımı için daha zor olur. Zincir kuralına göre, $t < T$ herhangi bir zamanda $\partial L/\partial \mathbf{h}_t \in \mathbb{R}^h$ gizli durumunun degradyanı olarak yeniden hesaplanabilir:

$$\frac{\partial L}{\partial \mathbf{h}_t} = \text{prod}\left(\frac{\partial L}{\partial \mathbf{h}_{t+1}}, \frac{\partial \mathbf{h}_{t+1}}{\partial \mathbf{h}_t} \right) + \text{prod}\left(\frac{\partial L}{\partial \mathbf{o}_t}, \frac{\partial \mathbf{o}_t}{\partial \mathbf{h}_t} \right) = \mathbf{W}_{hh}^\top \frac{\partial L}{\partial \mathbf{h}_{t+1}} + \mathbf{W}_{qh}^\top \frac{\partial L}{\partial \mathbf{o}_t}.$$
:eqlabel:`eq_bptt_partial_L_ht_recur`

Analiz için, herhangi bir zaman adım $1 \leq t \leq T$ için tekrarlayan hesaplama genişleyen verir

$$\frac{\partial L}{\partial \mathbf{h}_t}= \sum_{i=t}^T {\left(\mathbf{W}_{hh}^\top\right)}^{T-i} \mathbf{W}_{qh}^\top \frac{\partial L}{\partial \mathbf{o}_{T+t-i}}.$$
:eqlabel:`eq_bptt_partial_L_ht`

:eqref:`eq_bptt_partial_L_ht`'ten bu basit doğrusal örnek, uzun dizi modellerinin bazı temel problemlerini zaten sergilediğini görebiliyoruz: $\mathbf{W}_{hh}^\top$'nın potansiyel olarak çok büyük güçlerini içeriyor. İçinde, 1'den küçük özdeğerler kaybolur ve 1'den büyük özdeğerler sapar. Bu sayısal olarak kararsızdır, bu da kendini kaybolan ve patlayan degradeler şeklinde gösterir. Bunu ele almanın bir yolu, :numref:`subsec_bptt_analysis`'te ele alındığı gibi, zaman adımlarını hesaplama açısından uygun bir boyutta kesmektir. Pratikte, bu kesme, belirli bir sayıda zaman adımından sonra degradeyi ayırarak gerçekleştirilir. Daha sonra uzun kısa süreli bellek gibi daha sofistike dizi modellerinin bunu daha da hafifletebileceğini göreceğiz.

Son olarak, :numref:`fig_rnn_bptt` objektif fonksiyonun $L$ model parametrelerine bağlı olduğunu göstermektedir $\mathbf{W}_{hx}$ ve $\mathbf{W}_{hh}$ gizli katmanda $\mathbf{h}_1, \ldots, \mathbf{h}_T$. Bu tür parametrelere göre degradeleri hesaplamak için $\partial L / \partial \mathbf{W}_{hx} \in \mathbb{R}^{h \times d}$ ve $\partial L / \partial \mathbf{W}_{hh} \in \mathbb{R}^{h \times h}$,

$$
\begin{aligned}
\frac{\partial L}{\partial \mathbf{W}_{hx}}
&= \sum_{t=1}^T \text{prod}\left(\frac{\partial L}{\partial \mathbf{h}_t}, \frac{\partial \mathbf{h}_t}{\partial \mathbf{W}_{hx}}\right)
= \sum_{t=1}^T \frac{\partial L}{\partial \mathbf{h}_t} \mathbf{x}_t^\top,\\
\frac{\partial L}{\partial \mathbf{W}_{hh}}
&= \sum_{t=1}^T \text{prod}\left(\frac{\partial L}{\partial \mathbf{h}_t}, \frac{\partial \mathbf{h}_t}{\partial \mathbf{W}_{hh}}\right)
= \sum_{t=1}^T \frac{\partial L}{\partial \mathbf{h}_t} \mathbf{h}_{t-1}^\top,
\end{aligned}
$$

burada :eqref:`eq_bptt_partial_L_hT_final_step` ve :eqref:`eq_bptt_partial_L_ht_recur` tarafından yeniden hesaplanır $\partial L/\partial \mathbf{h}_t$ sayısal kararlılığı etkileyen anahtar miktardır.

Zaman içinde backpropagation RNN'lerde backpropagation uygulaması olduğundan, Biz açıkladığımız gibi :numref:`sec_backprop`, eğitim RNN zaman içinde geri yayılma ile ileri yayılım dönüşümlü. Ayrıca, zaman içinde geri yayılma hesaplar ve sırayla yukarıdaki degradeler depolar. Özellikle, depolanan ara değerler $\partial L / \partial \mathbf{W}_{hx}$ ve $\partial L / \partial \mathbf{W}_{hh}$ hesaplamalarında kullanılacak $\partial L/\partial \mathbf{h}_t$ depolama gibi yinelenen hesaplamaları önlemek için yeniden kullanılır.

## Özet

* Zamanda geriye yayılma, yalnızca gizli bir duruma sahip dizi modellerine geri yayılmanın bir uygulamasıdır.
* Düzenli kesme ve randomize kesme gibi hesaplama kolaylığı ve sayısal kararlılık için kesme gereklidir.
* Matrislerin yüksek güçleri farklı veya kaybolan özdeğerlere yol açabilir. Bu, patlayan veya kaybolan degradeler şeklinde kendini gösterir.
* Etkili hesaplama için ara değerler zaman içinde geri yayılım sırasında önbelleğe alınır.

## Egzersizler

1. Biz bir simetrik matris $\mathbf{M} \in \mathbb{R}^{n \times n}$ özdeğerler $\lambda_i$ olan karşılık gelen özvektörler $\mathbf{v}_i$ ($i = 1, \ldots, n$) olan varsayalım. Genellik kaybı olmadan, $|\lambda_i| \geq |\lambda_{i+1}|$ sırayla sipariş edildiklerini varsayalım.
   1. $\mathbf{M}^k$ özdeğerleri olduğunu göster $\lambda_i^k$.
   1. Rastgele bir vektör için $\mathbf{x} \in \mathbb{R}^n$, yüksek olasılıklı $\mathbf{M}^k \mathbf{x}$ çok fazla özvektör ile hizalanmış olacağını kanıtlayın $\mathbf{v}_1$
arasında $\mathbf{M}$. Bu ifadeyi resmileştir.
   1. Yukarıdaki sonuç RNN'lerdeki degradeler için ne anlama geliyor?
1. Degrade kırpmanın yanı sıra, tekrarlayan sinir ağlarında degrade patlaması ile başa çıkmak için başka yöntemler düşünebiliyor musunuz?

[Discussions](https://discuss.d2l.ai/t/334)