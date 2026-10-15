# Box_v2 — TikTok Lite Phone workflow

`Box_v2` là project GPM Automate loại **Phone** cho thao tác xem feed TikTok Lite. Tài liệu này phản ánh source hiện tại và kết quả vận hành đã được người dùng xác nhận trên Device 05 ngày 15-10-2026.

## Phạm vi hiện tại

| Thành phần | Vai trò | Trạng thái |
| --- | --- | --- |
| `flow_0_v2.gscript` | Mở TikTok Lite qua hai biến thể package/Activity và chờ ngắn | Có sẵn, chạy độc lập |
| `flow_1.gscript` | Chờ feed, xem video, vuốt 5 lượt, chọn ngẫu nhiên Like | Đã chạy thật thành công |
| `src.gscript` | Main workflow | Hiện chỉ gọi `flow_1` |

`flow_1` vì vậy **giả định TikTok Lite đã ở feed**. Khi cần mở app trước, chạy `flow_0_v2` riêng, hoặc chủ động nối lời gọi flow này vào Main trong một thay đổi có kiểm thử riêng.

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

## Các giới hạn đã chốt

- Không xác minh `Liked` sau khi chạm. XPath trạng thái Liked không ổn định; Image Search cũng có thể khớp sai do biểu tượng nằm trên nền video động.
- `tiktok_like.img64`, `tiktok_liked.img64`, và `tiktok_comment.img64` được giữ làm asset probe lịch sử. `flow_1` hiện không phụ thuộc các ảnh này để quyết định Like.
- Tọa độ Like đã xác nhận cho bố cục Device 05 / `1440 × 2960`. Cơ chế vuốt đã tỷ lệ theo kích thước màn hình; điểm Like chưa tỷ lệ theo màn hình và cần canary riêng trước khi dùng cho bố cục khác.
- Nếu video hiện tại đã Like, một lần chạm có thể đảo trạng thái. Flow không có cách xác nhận trạng thái trước/sau chạm đáng tin cậy.

## Cách chạy

1. Mở TikTok Lite đến feed **For You**, hoặc chạy `flow_0_v2` trước.
2. Sau khi sửa trực tiếp file `.gscript`, đóng/mở lại GPM Editor để nạp source mới.
3. Chạy Main (`src.gscript`) hoặc gọi `flow_1`.
4. Kiểm tra log: XPath `For You`, `$screenSizeRaw` (nếu có), các biến tọa độ vuốt, `$actionRoll`, rồi `$likeResult`.

Một action có dấu `✓` chỉ cho biết GPM Runtime đã thực thi node; `$likeResult` chỉ cho biết nhánh đã chọn. Không coi các log này là bằng chứng TikTok đã giữ trạng thái Like.

## Kiểm tra source tĩnh

```powershell
Get-ChildItem 'D:\gpm\Box_v2' -Filter *.gscript | ForEach-Object {
  Get-Content $_.FullName -Raw | ConvertFrom-Json | Out-Null
  "JSON OK: $($_.Name)"
}
```

Chi tiết node, tỉ lệ vuốt, launcher và checklist được ghi trong [PHONE_ACTION_RUNBOOK.md](PHONE_ACTION_RUNBOOK.md). Quan hệ Runtime/thiết bị/asset nằm trong [CONNECTIONS.md](CONNECTIONS.md).
