# 情報通信工学まとめ

# 1. 信号とシステム
- 信号のエネルギー

$$ E_g = \int_{-\infty}^{\infty} |g(t)|^2 dt $$

- 信号の電力

$$ P_g = \lim_{T \to \infty} \frac{1}{T} \int_{-T/2}^{T/2} |g(t)|^2 dt $$

- フーリエ級数展開
  $$ g(t) = \sum_{n=-\infty}^{\infty} c_n e^{j2\pi f_0 n t} $$

  $$ c_n = \frac{1}{2\pi}\int_{-\pi}^{\pi}g(t)e^{-j2\pi f_0nt} dt $$

- フーリエ変換
  $$ G(f) = \frac{1}{2\pi}\int_{-\infty}^{\infty} g(t) e^{-j2\pi ft} dt $$

    $$ g(t) = \int_{-\infty}^{\infty} G(f) e^{j2\pi ft} df $$

# 2. 信号の分析と伝送

# 3. アナログ変調

## 1. 抑圧搬送波両側波帯（Double Sideband Suprressed Carrier, DSB-SC）

$$ x(t) = m(t)\cos(\omega_c t+ \phi) $$

のように、 $\cos$ で変調する方法。以下、 $\phi = 0$ とする。 

フーリエ変換は

$$ X(f) = \frac{1}{2} (M(f+f_c)+ M(f-f_c))$$

のように、2つに分かれて振幅半分となる。

USB ... upper side band
LSB ... lower side band

- 復調方法
  -  $ \cos(2\pi f_c t) $ をかけてLPFに通して行う。
  $$ e(t) = x(t)\cos(2 \pi f_c t) = m(t) \cos^2(2\pi f_c t) = \frac{1}{2}m(t) + \frac{1}{2}\cos(4\pi f_c t) $$

  