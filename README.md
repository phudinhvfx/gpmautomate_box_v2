# Box_v2 — TikTok Lite Phone workflow

`Box_v2` là project GPM Automate loại **Phone** cho thao tác TikTok Lite. Tài liệu này phản ánh source hiện tại và bốn flow đã được người vận hành kiểm thử độc lập ngày 16-10-2026.

## Phạm vi hiện tại

| Thành phần | Vai trò | Trạng thái |
| --- | --- | --- |
| `flow_0_v2.gscript` | Mở TikTok Lite qua hai biến thể package/Activity, rồi xác minh **For You** | Đã kiểm thử độc lập |
| `flow_1.gscript` | Chờ feed, xem video, vuốt 5 lượt, chọn ngẫu nhiên Like | Đã kiểm thử độc lập |
| `flow_2.gscript` | Chờ feed, mở Comment, nhập/gửi text, xác minh gửi rồi đóng panel | Đã kiểm thử độc lập |
| `flow_3.gscript` | Chờ feed, mở Comment, đọc/scroll lên rồi đóng panel | Đã kiểm thử độc lập |
| `src.gscript` | Main workflow | Hiện gọi `flow_2` |

`flow_1`, `flow_2` và `flow_3` đều **giả định TikTok Lite đã ở feed**. Khi cần mở app trước, chạy `flow_0_v2` riêng, hoặc chủ động nối lời gọi flow này vào Main trong một thay đổi có kiểm thử riêng.

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

## Hành vi của `flow_2`: gửi comment

1. Chờ **For You**, đọc `adb shell wm size` (fallback `1440 × 2960`), xem video hiện tại rồi vuốt lên 3 video mới theo tọa độ tỷ lệ màn hình.
2. `$commentRoll` hiện là **1–2**, nên luôn vào nhánh Comment. Flow chạm `1337,1897` với jitter `±15 px`, chờ 8–12 giây và xác minh panel bằng `tiktok_verifypanelcomment.img64`.
3. Tìm `tiktok_addcomment.img64`, chạm ô nhập, chờ 6–8 giây, đọc một dòng ngẫu nhiên từ `D:\docs\comments_warmup.telex.txt` và nhập Telex.
4. Tìm `tiktok_sendcomment.img64`, chạm Send, chờ 5–8 giây rồi xác minh `tiktok_verifysentcomment.img64`.
5. Khi Send đã tìm được, chờ thêm 3 giây, tìm/chạm `tiktok_closecomment.img64`, chờ 3 giây và xác minh panel không còn hiện.

`$flow2Result` giữ kết quả gửi: `COMMENT_SEND_VERIFIED`, `COMMENT_SEND_TAPPED_NOT_VERIFIED`, hoặc `COMMENT_SEND_BUTTON_NOT_FOUND`. Biến `$commentPanelCloseResult` ghi riêng kết quả đóng: `COMMENT_PANEL_CLOSED`, `COMMENT_PANEL_STILL_OPEN`, hoặc `COMMENT_CLOSE_ICON_NOT_FOUND`.

## Hành vi của `flow_3`: đọc comment

1. Dùng cùng gate **For You**, kích thước màn hình, ba lượt vuốt video và chạm Comment có jitter như `flow_2`; thời gian chờ panel là 6–10 giây.
2. Sau khi xác minh panel, flow chỉ vuốt **lên** trong vùng comment 2–4 lượt. Không có vuốt xuống để tránh chuyển video hoặc kích hoạt ô nhập.
3. Flow chờ 2 giây, tìm/chạm Close, rồi xác minh panel. Nếu panel vẫn còn (ví dụ Close đầu chỉ đóng bàn phím ảo), flow tìm/chạm Close thêm lần nữa và xác minh lại.

`flow_3` kế thừa các mã `COMMENT_PANEL_*` lịch sử trong `$flow2Result`; tên biến này không làm thay đổi hành vi của Flow 3.

## Các giới hạn đã chốt

- Không xác minh `Liked` sau khi chạm. XPath trạng thái Liked không ổn định; Image Search cũng có thể khớp sai do biểu tượng nằm trên nền video động.
- `tiktok_like.img64`, `tiktok_liked.img64`, và `tiktok_comment.img64` được giữ làm asset probe lịch sử. `flow_1` hiện không phụ thuộc các ảnh này để quyết định Like.
- `flow_2` dùng asset hình ảnh cho ô nhập, Send, trạng thái đã gửi, panel và Close; `flow_3` dùng asset panel và Close. Các asset này cần canary lại nếu layout TikTok Lite đổi.
- Tọa độ Like đã xác nhận cho bố cục Device 05 / `1440 × 2960`. Cơ chế vuốt đã tỷ lệ theo kích thước màn hình; điểm Like chưa tỷ lệ theo màn hình và cần canary riêng trước khi dùng cho bố cục khác.
- Nếu video hiện tại đã Like, một lần chạm có thể đảo trạng thái. Flow không có cách xác nhận trạng thái trước/sau chạm đáng tin cậy.

## Cách chạy

1. Mở TikTok Lite đến feed **For You**, hoặc chạy `flow_0_v2` trước.
2. Sau khi sửa trực tiếp file `.gscript`, đóng/mở lại GPM Editor để nạp source mới.
3. Chạy Main (`src.gscript`) hoặc gọi riêng flow cần kiểm thử: `flow_1` cho Like, `flow_2` cho gửi comment, `flow_3` cho đọc comment.
4. Với Main/`flow_2`, kiểm tra XPath `For You`, `$screenSizeRaw` (nếu có), các biến tọa độ, `$commentRoll`, `$commentPos`, `$flow2Result` và `$commentPanelCloseResult`. Với `flow_3`, kiểm tra `$flow2Result` sau thao tác đọc/đóng.
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
