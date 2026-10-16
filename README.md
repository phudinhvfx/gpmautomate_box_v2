# Box_v2 — TikTok Lite Phone workflow

`Box_v2` là project GPM Automate loại **Phone** cho thao tác xem feed TikTok Lite. Tài liệu này phản ánh source hiện tại và kết quả vận hành đã được người dùng xác nhận trên Device 05 ngày 15-10-2026.

## Phạm vi hiện tại

| Thành phần | Vai trò | Trạng thái |
| --- | --- | --- |
| `flow_0_v2.gscript` | Mở TikTok Lite qua hai biến thể package/Activity và chờ ngắn | Có sẵn, chạy độc lập |
| `flow_1.gscript` | Chờ feed, xem video, vuốt 5 lượt, chọn ngẫu nhiên Like | Đã chạy thật thành công |
| `flow_2.gscript` | Chờ feed, xem/đổi 3 video, rồi đọc và đóng Comment panel | Đã được người vận hành kiểm thử thực tế |
| `src.gscript` | Main workflow | Hiện gọi `flow_2` |

`flow_1` và `flow_2` đều **giả định TikTok Lite đã ở feed**. Khi cần mở app trước, chạy `flow_0_v2` riêng, hoặc chủ động nối lời gọi flow này vào Main trong một thay đổi có kiểm thử riêng.

## Hành vi của `flow_1`

1. Chờ XPath `//node[@text="For You" and @class="android.widget.Button"]` để xác nhận feed.
2. Đọc kích thước màn hình bằng `adb shell wm size`; nếu Runtime không trả được kết quả, dùng fallback `1440 × 2960` (Device 05).
3. Xem video đang hiển thị ngẫu nhiên 3–8 giây.
4. Vuốt lên `5` lượt; mỗi lượt chọn tọa độ theo tỷ lệ màn hình, chờ video mới và xem tiếp 3–8 giây.
5. Random số từ 1 đến 9. Chỉ số `1` hoặc `2` mới gửi thao tác Like; các số khác ghi log bỏ qua.
6. Like tại tâm đã kiểm thử trên Device 05 là `1333,1672`, có jitter ngẫu nhiên `±8 px` cho từng lần chạm.

Kết quả nhánh được ghi trong `$likeResult`:

| Giá trị | Ý nghĩa |
| --- | --- |
| `LIKE_TAP_SENT_ROLL_<n>` | Đã gửi chạm Like khi lượt random là 1 hoặc 2 |
| `LIKE_NOT_SELECTED_ROLL_<n>` | Không chọn Like khi lượt random là 3–9 |

Đây là log của **ý định/thao tác đã gửi**, không phải chứng minh trạng thái Like phía TikTok.

## Hành vi của `flow_2`

1. Chờ XPath **For You**, đọc `adb shell wm size`, rồi dùng fallback `1440 × 2960` nếu không đọc được kích thước thật.
2. Xem video hiện tại 3–8 giây, sau đó vuốt lên 3 video mới bằng tọa độ tỷ lệ màn hình; mỗi video mới cũng chờ 3–8 giây.
3. Sinh `$commentRoll`. Source hiện đặt khoảng **1–2**, vì vậy nhánh đọc comment hiện luôn được chọn. Nhánh log `RANDOM_NUMBER_<n>_NO_READ_COMMENT` chỉ có ý nghĩa khi khoảng random được thay đổi trong một thay đổi riêng.
4. Chạm nút Comment tại điểm đã kiểm thử `1337,1897`, với jitter độc lập `±15 px`; chờ 6–10 giây và xác minh panel bằng `tiktok_verifypanelcomment.img64`.
5. Khi panel đã xác minh, chỉ vuốt **lên** trong vùng comment 2–4 lượt, rồi chờ 2 giây trước khi đóng. Không có vuốt xuống: vuốt đúng hướng/tọa độ là điều kiện giữ thao tác nằm trong panel thay vì đổi video.
6. Tìm `tiktok_closecomment.img64`, chạm Close, chờ 3 giây và tìm lại ảnh panel. Nếu panel vẫn còn (ví dụ lần chạm đầu chỉ đóng bàn phím ảo của bài không có comment), flow tìm Close và chạm thêm **một lần**, rồi xác minh lại sau 3 giây.

`flow_2` chỉ báo trạng thái thao tác đã gửi hoặc trạng thái Image Search. Kết quả cần được đối chiếu với quan sát UI khi canary một layout/app version mới.

Các giá trị `$flow2Result` quan trọng:

| Giá trị | Ý nghĩa |
| --- | --- |
| `COMMENT_TAP_TARGET_<x,y>` | Đã tạo tọa độ chạm Comment có jitter |
| `COMMENT_PANEL_OPEN` | Asset panel đã được tìm thấy sau khi mở |
| `COMMENT_PANEL_CLOSED` | Panel không còn thấy sau lần Close đầu |
| `COMMENT_PANEL_CLOSED_AFTER_RETRY` | Panel không còn thấy sau lần Close thứ hai |
| `COMMENT_PANEL_STILL_OPEN_AFTER_RETRY` | Panel vẫn thấy sau hai lần Close |
| `COMMENT_CLOSE_ICON_NOT_FOUND` / `COMMENT_CLOSE_RETRY_ICON_NOT_FOUND` | Không tìm được Close ở lần tương ứng |
| `COMMENT_PANEL_NOT_VERIFIED` | Không xác minh được panel sau khi chạm Comment |

## Các giới hạn đã chốt

- Không xác minh `Liked` sau khi chạm. XPath trạng thái Liked không ổn định; Image Search cũng có thể khớp sai do biểu tượng nằm trên nền video động.
- `tiktok_like.img64`, `tiktok_liked.img64`, và `tiktok_comment.img64` được giữ làm asset probe lịch sử. `flow_1` hiện không phụ thuộc các ảnh này để quyết định Like.
- Tọa độ Like đã xác nhận cho bố cục Device 05 / `1440 × 2960`. Cơ chế vuốt đã tỷ lệ theo kích thước màn hình; điểm Like chưa tỷ lệ theo màn hình và cần canary riêng trước khi dùng cho bố cục khác.
- Nếu video hiện tại đã Like, một lần chạm có thể đảo trạng thái. Flow không có cách xác nhận trạng thái trước/sau chạm đáng tin cậy.

## Cách chạy

1. Mở TikTok Lite đến feed **For You**, hoặc chạy `flow_0_v2` trước.
2. Sau khi sửa trực tiếp file `.gscript`, đóng/mở lại GPM Editor để nạp source mới.
3. Chạy Main (`src.gscript`) hoặc gọi riêng flow cần kiểm thử: `flow_1` cho Like, `flow_2` cho đọc comment.
4. Với Main/`flow_2`, kiểm tra XPath `For You`, `$screenSizeRaw` (nếu có), các biến tọa độ vuốt, `$commentRoll`, `$commentPos`, rồi `$flow2Result`.
5. Với `flow_1`, kiểm tra các biến tọa độ vuốt, `$actionRoll`, rồi `$likeResult`.

Một action có dấu `✓` chỉ cho biết GPM Runtime đã thực thi node; `$likeResult` chỉ cho biết nhánh đã chọn. Không coi các log này là bằng chứng TikTok đã giữ trạng thái Like.

## Kiểm tra source tĩnh

```powershell
Get-ChildItem 'D:\gpm\Box_v2' -Filter *.gscript | ForEach-Object {
  Get-Content $_.FullName -Raw | ConvertFrom-Json | Out-Null
  "JSON OK: $($_.Name)"
}
```

Chi tiết node, tỉ lệ vuốt, launcher và checklist được ghi trong [PHONE_ACTION_RUNBOOK.md](PHONE_ACTION_RUNBOOK.md). Quan hệ Runtime/thiết bị/asset nằm trong [CONNECTIONS.md](CONNECTIONS.md).
