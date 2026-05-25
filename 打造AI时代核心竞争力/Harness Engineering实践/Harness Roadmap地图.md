



使用rust的clippy检测项目代码的圈复杂度和认知复杂度：

```
root@ebpf-machine:/home/work/suricata-8.0.4-study/rust# cargo clippy -p suricata --lib --no-deps
    Checking suricata-htp v8.0.4 (/home/work/suricata-8.0.4-study/rust/htp)
   Compiling suricata-derive v8.0.4 (/home/work/suricata-8.0.4-study/rust/derive)
    Checking suricata-sys v8.0.4 (/home/work/suricata-8.0.4-study/rust/sys)
    Checking suricata v8.0.4 (/home/work/suricata-8.0.4-study/rust)
warning: very complex type used. Consider factoring parts into `type` definitions
  --> src/iec61850mms/ber.rs:53:46
   |
53 | pub(super) fn parse_ber_tlv(input: &[u8]) -> Result<(u8, bool, u32, &[u8], &[u8]), ()> {
   |                                              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = help: for further information visit https://rust-lang.github.io/rust-clippy/rust-1.91.0/index.html#type_complexity
   = note: `#[warn(clippy::type_complexity)]` on by default
```

上文中报parse_ber_tlv函数的复杂度太高，考虑优化下。

我问了下AI工具，推荐改为

```
pub(super) struct BerTlv<'a> {
      pub tag: u8,
      pub constructed: bool,
      pub length: u32,
      pub value: &'a [u8],
      pub rest: &'a [u8],
  }

  pub(super) fn parse_ber_tlv(input: &[u8]) -> Result<BerTlv<'_>, ()> {
      ...
  }

  调用方会更清楚：

  let tlv = parse_ber_tlv(input)?;
  if tlv.tag == 0xa0 {
      ...
  }
```



如何重新获得项目代码的掌控感？

先做一份 iec61850mms 代码地图文档，不改代码，只整理调用链和模块职责。