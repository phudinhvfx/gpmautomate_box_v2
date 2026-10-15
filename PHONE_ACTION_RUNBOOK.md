# Runbook — Box_v2 TikTok Lite

## Điều kiện và giới hạn

| Mục | Giá trị hiện tại |
| --- | --- |
| Project | `D:\gpm\Box_v2` |
| Loại project | GPM Automate Phone |
| Feed ổn định | `For You` — XPath `//node[@text="For You" and @class="android.widget.Button"]` |
| Thiết bị đã canary | Device 05, bố cục `1440 × 2960` |
| Main hiện tại | Chỉ gọi `flow_1` |
| Số lượt vuốt mặc định | `5` |
| Cơ hội Like | 2/9 (khi `$actionRoll` là 1 hoặc 2) |

Không chạy GPM Phone đồng thời với một workflow BoxPhone V3 khác trên cùng device. Việc chọn device/user/proxy nằm ngoài phạm vi của project này.

```mermaid
flowchart TD
  A[flow_0_v2 chạy độc lập] --> B[TikTok Lite foreground]
  C[Main src.gscript] --> D[flow_1]
  B --> D
  D --> E[Chờ XPath For You]
  E --> F[Đọc wm size / fallback]
  F --> G[Xem 3-8 giây]
  G --> H[Vuốt lên 5 lượt, mỗi lượt xem 3-8 giây]
  H --> I[Random 1-9]
  I -->|1 hoặc 2| J[Chạm Like với jitter]
  I -->|3-9| K[Ghi LIKE_NOT_SELECTED]
  J --> L[Ghi LIKE_TAP_SENT]
```

## `flow_0_v2`: mở TikTok Lite

Flow này không được Main gọi mặc định. Nó thử các launcher theo thứ tự sau:

1. Legacy: `com.zhiliaoapp.musically.go/.mini.MainActivity`.
2. Đọc `dumpsys activity activities`, xác nhận `topResumedActivity` có package legacy.
3. Nếu chưa foreground, thử modern: `com.tiktok.lite.go/com.zhiliaoapp.musically.go.mini.MainActivity`.
4. Xác nhận foreground package modern và chờ 1,5–2,5 giây.

Không thêm `--user` vào lệnh launcher/package. Runtime từng trả `Bad user number` khi nhận Android user `10`; launcher không có `--user` đã mở thành công TikTok Lite trên Device 05. Việc thành công từ `am start` phải được đối chiếu bằng bước foreground, không chỉ dựa vào dấu `✓` của action.

## `flow_1`: xem, vuốt và Like xác suất

### 1. Xác nhận feed và kích thước màn hình

- `PhoneWaitElement` chờ **For You** tối đa 20 giây.
- `PhoneExecuteAdb` gọi `shell wm size`, lưu vào `$screenSizeRaw`.
- Hai node Regex lấy `$screenWidthRead` và `$screenHeightRead` từ dòng `Physical size: WIDTHxHEIGHT`.
- Nếu thiếu một trong hai giá trị, flow giữ fallback `$screenWidth = 1440`, `$screenHeight = 2960`.

Fallback là tương thích Device 05, không phải khẳng định mọi device có cùng kích thước. Luôn xem log kích thước khi canary một bố cục mới.

### 2. Vùng vuốt theo tỷ lệ

Mỗi giá trị unit được random rồi quy đổi bằng `dimension × unit / 20`.

| Điểm | Unit | Tỷ lệ màn hình | Mục đích |
| --- | ---: | ---: | --- |
| Start X | 8–12 | 40–60% chiều rộng | Tránh mép trái/phải |
| Start Y | 16–17 | 80–85% chiều cao | Bắt đầu thấp trên video |
| End X | 8–12 | 40–60% chiều rộng | Giữ đường vuốt tự nhiên |
| End Y | 8–9 | 40–45% chiều cao | Kết thúc vùng giữa trên |
| Duration | 450–700 ms | — | Nhịp vuốt |

Mỗi lượt:

1. Sinh unit và duration.
2. Tính `$scrollStartX/Y`, `$scrollEndX/Y`, rồi ghép thành `$scrollStartPos` và `$scrollEndPos`.
3. `PhoneScroll` với `SCROLL_TYPE=POINTS`, `DIRECTION=Up`.
4. Chờ/xem video mới ngẫu nhiên 3–8 giây.

Flow cũng chờ 3–8 giây trước lượt vuốt đầu tiên. Vòng lặp dùng `$swipeCount = 5`; cần đổi biến này khi muốn thay số lượt, không sửa rời từng node.

### 3. Quyết định Like

Sau lượt vuốt cuối:

1. `NumberRandom` ghi `$actionRoll` trong khoảng 1–9.
2. Điều kiện `$actionRoll = 1 || $actionRoll = 2` đi vào nhánh Like.
3. Nhánh Like sinh `$likeJitterX/Y` trong khoảng -8…8 và tính:

   ```text
   likeTapX = 1333 + likeJitterX
   likeTapY = 1672 + likeJitterY
   ```

4. `PhoneTouch` chạm `$likeTapPos`, sau đó ghi `LIKE_TAP_SENT_ROLL_<n>`.
5. Nhánh còn lại không chạm màn hình, ghi `LIKE_NOT_SELECTED_ROLL_<n>`.

Không đặt Delay 5 giây hay Image Search để xác minh Liked sau chạm: ảnh trái tim đỏ không cho kết quả đủ ổn định trên nền video thay đổi. Điều này cố ý; không diễn giải `LIKE_TAP_SENT` là Like đã được TikTok ghi nhận.

## Bằng chứng hiện có

| Hạng mục | Kết quả |
| --- | --- |
| Mở Lite legacy trên Device 05 | `topResumedActivity` xác nhận `com.zhiliaoapp.musically.go/.mini.MainActivity` foreground |
| XPath For You | Đã khớp trong các lần probe trên Device 05 |
| Vuốt tỷ lệ/ngẫu nhiên | Đã chạy trong probe; `PhoneScroll` chuyển video |
| Like fixed-point | Quan sát thực tế xác nhận thao tác Like tại vùng `1333,1672` trên Device 05 |
| `flow_1` hiện tại | Người vận hành xác nhận chạy thật thành công |
| Xác minh Liked bằng XPath/ảnh | Không đáng tin cậy, nên đã loại khỏi flow chính |

## Checklist mỗi lần canary

1. TikTok Lite đã mở, mạng hoạt động và đang ở feed **For You**.
2. Đảm bảo editor đã nạp source hiện tại.
3. Xác nhận log `$screenSizeRaw`; nếu thiếu, xác nhận thiết bị thực sự dùng fallback `1440 × 2960` hoặc dừng để đánh giá bố cục.
4. Xem các `$scrollStartPos`, `$scrollEndPos` không chạm khu vực điều hướng/bottom bar.
5. Ghi nhận `$actionRoll` và `$likeResult`. Lượt 3–9 không Like là hành vi đúng.
6. Với lượt 1–2, chỉ đánh giá bằng quan sát UI nếu cần; không lấy Image Search Liked làm bằng chứng pass/fail.

Nếu XPath **For You** timeout, dừng flow và khôi phục app/feed trước khi chạy lại. Không thêm click mù hoặc selector suy đoán vào flow này.
