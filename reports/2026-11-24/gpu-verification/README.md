# VIctory GPU verification, November 24, 2026

Original hardware testing and measurements by **YUNISKH (@yuniskh)**. I used Codex to prepare the regression tests, run them, and check the logs. All images were generated from test patterns.

## Recording order and boxed copies

I ran my original two-list fixture on Windows using an RTX 5090 and Microsoft WARP. With both copies submitted together in A/B order, upstream `ccfb029` preserved seven of sixteen pixels without a separate setup upload. All nine untouched pixels read back as zero. Preloading the destination preserved all sixteen. On @zondy1i’s PR #1934, all eight cases preserved all sixteen pixels on both adapters. The eight cases vary source type, recording order and setup upload. I ran the PR at `5fb9f47`. Every tracked file and mode matched the later revision `afd60ab`.

[My original issue #1891](https://github.com/Marvisak/VIctory/issues/1891) · [Original two-list measurements](https://github.com/Marvisak/VIctory/issues/49#issuecomment-7148189167) · [PR #1934](https://github.com/Marvisak/VIctory/pull/1934)

The smaller regression takes the middle 2×2 box from a 4×4 buffer footprint and copies it to the top-left of a 4×4 texture. The twelve other pixels were initialized to `DB DB DB DB`. The source and destination have the same extent and the destination offset is zero. The source box is the only indication that the write is partial.

| Revision or control | RTX 5090 | WARP |
| --- | --- | --- |
| Upstream `940d32f` with the new test | 4/16 correct | 4/16 correct |
| PR #1934 with the same test | 16/16 | 16/16 |
| Local control that ignores the box only when deciding whether to preserve the destination | 4/16 | 4/16 |

The whole-buffer copy test passed in every run. PR #1934’s existing submission-order test passed in both the fixed build and the control. With the new test and PR #1934 together, the diagnostics GPU suite on WARP passed 146 tests, with 2 ignored. The regression branch changes test code only. It is intended to land with or after the production fix.

[Boxed-copy test commit on my fork](https://github.com/yuniskh/VIctory/commit/d6f53898968e94a8afd3f7c1a2574a8122183011) · [Production fix by @zondy1i](https://github.com/Marvisak/VIctory/pull/1934)

## NV12 and P010 footprints

The footprint regression failed on `ccfb029`, which returned zero widths and heights for a valid two-slice NV12 array. With the change, the public query returned both full-size luma planes followed by the two half-size chroma planes. The test also starts a query at the first chroma slice and checks the data size including its unpadded final row.

I compared VIctory to native D3D12 for both formats, using six image sizes from 2×2 to 3840×2160, one to three array slices, every contiguous subresource range, and base offsets 0, 512 and 4096. Each query checked the total size and each returned plane’s offset, format, width, height, depth, row pitch, row count and unpadded row size.

| Adapter | Queries matched | Plane entries matched |
| --- | ---: | ---: |
| NVIDIA GeForce RTX 5090 | 1,224 / 1,224 | 2,880 / 2,880 |
| Integrated AMD Radeon Graphics | 1,224 / 1,224 | 2,880 / 2,880 |

Both physical GPUs are in the same desktop. WARP is the separate software control for recording order. The video-format comparison used aligned base offsets. The shared calculation for unaligned base offsets has its own PR. This work builds on @broccolize’s existing depth/stencil plane handling. @ToxicmaggotRP’s PR covers packed RGB and the shared base-offset calculation.

On the video branch, the WARP diagnostics suite passed 145 tests with 2 ignored. The default hardware suite passed 140, failed 2 and ignored 2. The same descriptor-heap tests failed on the unpatched baseline. Applying the video change to newer upstream `940d32f` gave 147 passed and 2 ignored on WARP, and 142 passed, 2 failed and 2 ignored on the default hardware suite. The two failing tests were `dynamic_slots_are_held_until_released` and `lists_share_a_table_and_copies_keep_their_views`. Clippy for `victory-gpu` passed with default and all features on the video branch, and the asset-policy check passed.

[Original two-GPU video measurements](https://github.com/Marvisak/VIctory/issues/386#issuecomment-7120877314) · [Video change and portable regression on my fork](https://github.com/yuniskh/VIctory/commit/0477a4b6472e93d932d9bf21a91c3b6c0920c6a2) · [Depth/stencil PR #504](https://github.com/Marvisak/VIctory/pull/504) · [Packed-format and base-offset PR #409](https://github.com/Marvisak/VIctory/pull/409)

## Reproduce on Windows

Use a separate clean checkout and build directory for each revision. The commands use the project’s pinned MSVC Rust toolchain. Download the matching text patch from this folder and give `git apply` its local path. The patches contain only tests. [Results excerpt](results.txt)

**Original two-list fixture.** Use upstream `ccfb02996eac831853e8324156f123eeff083e9d` or PR #1934 at `5fb9f47a55eedd633e80c3b0f53d861414efad33`. The test prints and checks which adapter it used. [Recording-order test patch](recording-order.patch.txt)

```powershell
git apply C:/tests/recording-order.patch.txt
$env:CARGO_TARGET_DIR = Join-Path $PWD 'target'
$env:VICTORY_D3D12_WARP = '0'
$env:VICTORY_TEST_EXPECT_ADAPTER = 'RTX 5090'
cargo test --locked -p victory-gpu --lib --features diagnostics resource::tests::copy_recording_order_1891 -- --exact --ignored --nocapture --test-threads=1
$env:VICTORY_D3D12_WARP = '1'
$env:VICTORY_TEST_EXPECT_ADAPTER = 'Microsoft'
cargo test --locked -p victory-gpu --lib --features diagnostics resource::tests::copy_recording_order_1891 -- --exact --ignored --nocapture --test-threads=1
```

**Boxed-copy test.** Run the command below on the boxed-copy branch for the baseline. For the fixed build, check out PR #1934 in a fresh checkout, run `git apply C:/tests/boxed-copy.patch.txt`, and repeat the command. Set `VICTORY_D3D12_WARP` to `0` for hardware or `1` for WARP. [Boxed-copy test patch](boxed-copy.patch.txt)

```powershell
cargo test --locked -p victory-gpu --lib --features diagnostics resource::tests::boxed_buffer_copy_at_origin_preserves_untouched_texels -- --exact --test-threads=1
```

**Native video comparison.** Use the video commit linked above. The portable regression runs without a GPU. The optional comparison enumerates D3D12 hardware adapters on the machine and checks the results against each device. [Native video comparison patch](native-video-planes.patch.txt)

```powershell
cargo test --locked -p victory-gpu --lib resource::tests::planar_video_copy_footprints -- --exact
git apply C:/tests/native-video-planes.patch.txt
cargo test --locked -p victory-gpu --lib resource::tests::compare_video_layouts_to_native -- --exact --ignored --nocapture --test-threads=1
```
