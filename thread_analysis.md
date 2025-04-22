```
// 数据校验分支
void verify_data(uint8_t *data) {
    uint8x16_t simd_data = vld1q_u8(data);
    uint8x16_t simd_mask = vceqq_u8(simd_data, vdupq_n_u8(0));
    if (vgetq_lane_u64(vreinterpretq_u64_u8(simd_mask), 0) == 0) {
        // 推测执行中 SIMD 比较被跳过，分支误判进入敏感路径
        log_error(data);  // 可能触发越界内存访问
    }
}
```