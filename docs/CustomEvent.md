# Custom event thực sự trên DOM (CustomEvent)


## Detail
+ Đây là event native của browser, bạn tạo bằng new CustomEvent() và lắng nghe bằng addEventListener.
+ React có thể tương tác với nó thông qua useEffect.

```reacjs
// Component phát sự kiện
export function EventSender() {
  const sendEvent = () => {
    const event = new CustomEvent("myCustomEvent", {
      detail: { message: "Hello from EventSender" }
    });
    window.dispatchEvent(event);
  };

  return <button onClick={sendEvent}>Gửi Custom DOM Event</button>;
}

```

```reacjs
// Component lắng nghe sự kiện
import { useEffect } from "react";

export function EventListener() {
  useEffect(() => {
    const handler = (e) => {
      console.log("Nhận custom event:", e.detail);
    };
    window.addEventListener("myCustomEvent", handler);

    return () => {
      window.removeEventListener("myCustomEvent", handler);
    };
  }, []);

  return <div>Lắng nghe custom event...</div>;
}

```

## Case study
### Refresh token cross microsite
+ Để bài:
```
Triển khai refresh token cho nhiều site khác nhau trong reactjs.
bProWeb con host chứa widget.
1. Khi Widget  gọi API với token cũ → bị 401.
2. Widget bắn requestTokenRefresh qua window event.
3. bProWeb bắt được → gọi API refresh token → bắn tokenRefreshed với token mới.
4. Widget nhận token mới và gọi API lại
Widget sử dụng axios intercepter đển handler 401 và sẽ có timeout đợi refresh token 1s không có thì trả về lỗi và dựng lại. 
Dùng customEvent để triển khai giúp tôi giải pháp trên.
```

## Docs
1. https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent/CustomEvent
