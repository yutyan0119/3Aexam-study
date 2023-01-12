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
