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
+ Nguyên tắc hoạt động
```
+ Widget khi API trả về 401 → dispatch sự kiện "requestTokenRefresh".
+ bProWeb (host) lắng nghe "requestTokenRefresh" → gọi API refresh token → dispatch "tokenRefreshed" kèm token mới.
+ Widget lắng nghe "tokenRefreshed" → cập nhật token → gọi lại API.
+ Nếu Widget đợi hơn 1 giây không nhận được token mới → trả lỗi và dựng lại.
```
+ Triển khai widget
```reactjs
// widgetAxios.js
import axios from "axios";

let accessToken = localStorage.getItem("accessToken");

// Hàm cập nhật token mới khi nhận được từ host
function setAccessToken(token) {
  accessToken = token;
  localStorage.setItem("accessToken", token);
}

// Lắng nghe sự kiện từ bProWeb
window.addEventListener("tokenRefreshed", (event) => {
  const { token } = event.detail;
  if (token) {
    setAccessToken(token);
    console.log("Widget: Token mới nhận:", token);
  }
});

const widgetAxios = axios.create({
  baseURL: "https://api.example.com",
});

widgetAxios.interceptors.request.use((config) => {
  if (accessToken) {
    config.headers.Authorization = `Bearer ${accessToken}`;
  }
  return config;
});

widgetAxios.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      console.log("Widget: Token hết hạn, yêu cầu refresh...");

      // Bắn sự kiện yêu cầu refresh
      window.dispatchEvent(new CustomEvent("requestTokenRefresh"));

      // Chờ tối đa 1 giây để nhận token mới
      const newToken = await new Promise((resolve) => {
        let resolved = false;

        const handler = (event) => {
          resolved = true;
          window.removeEventListener("tokenRefreshed", handler);
          resolve(event.detail.token);
        };

        window.addEventListener("tokenRefreshed", handler);

        // Timeout 1 giây
        setTimeout(() => {
          if (!resolved) {
            window.removeEventListener("tokenRefreshed", handler);
            resolve(null);
          }
        }, 1000);
      });

      if (newToken) {
        console.log("Widget: Đang gọi lại API với token mới...");
        return widgetAxios(error.config); // Gọi lại API
      } else {
        console.error("Widget: Không nhận được token mới, trả lỗi.");
        return Promise.reject(error);
      }
    }

    return Promise.reject(error);
  }
);

export default widgetAxios;

```
+ Triển khai bProWeb case detail
```reactjs
// widgetAxios.js
import axios from "axios";

let accessToken = localStorage.getItem("accessToken");

// Hàm cập nhật token mới khi nhận được từ host
function setAccessToken(token) {
  accessToken = token;
  localStorage.setItem("accessToken", token);
}

// Lắng nghe sự kiện từ bProWeb
window.addEventListener("tokenRefreshed", (event) => {
  const { token } = event.detail;
  if (token) {
    setAccessToken(token);
    console.log("Widget: Token mới nhận:", token);
  }
});

const widgetAxios = axios.create({
  baseURL: "https://api.example.com",
});

widgetAxios.interceptors.request.use((config) => {
  if (accessToken) {
    config.headers.Authorization = `Bearer ${accessToken}`;
  }
  return config;
});

widgetAxios.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      console.log("Widget: Token hết hạn, yêu cầu refresh...");

      // Bắn sự kiện yêu cầu refresh
      window.dispatchEvent(new CustomEvent("requestTokenRefresh"));

      // Chờ tối đa 1 giây để nhận token mới
      const newToken = await new Promise((resolve) => {
        let resolved = false;

        const handler = (event) => {
          resolved = true;
          window.removeEventListener("tokenRefreshed", handler);
          resolve(event.detail.token);
        };

        window.addEventListener("tokenRefreshed", handler);

        // Timeout 1 giây
        setTimeout(() => {
          if (!resolved) {
            window.removeEventListener("tokenRefreshed", handler);
            resolve(null);
          }
        }, 1000);
      });

      if (newToken) {
        console.log("Widget: Đang gọi lại API với token mới...");
        return widgetAxios(error.config); // Gọi lại API
      } else {
        console.error("Widget: Không nhận được token mới, trả lỗi.");
        return Promise.reject(error);
      }
    }

    return Promise.reject(error);
  }
);

export default widgetAxios;

```

## Docs
1. https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent/CustomEvent
