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

## Docs
1. https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent/CustomEvent
