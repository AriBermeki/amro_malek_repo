# amro_malek_repo

```rust
use std::{collections::HashMap, sync::Arc};
use tao::window::{Window, WindowId};
use wry::WebView;
use crate::{pipeevents::PythonEvent, pythonpipe::PythonCommands, rustpipe::RustCommands, utils::arc_mut};

#[allow(dead_code)]
pub(crate) async fn tcp_webview_connections(
    frompython: Arc<PythonCommands>, 
    fromrust: Arc<RustCommands>,
    webviews: &mut HashMap<WindowId, (Window, WebView)>,  // Change the key to WindowId
    id: WindowId
) {
    let _rust_engine = fromrust.clone();
    let python_icomming_event = frompython.clone();
    let webview_arc = arc_mut(webviews);

    while let Some(event) = python_icomming_event.receive_event().await {
        match event {
            PythonEvent::SetTitle(title) => {
                let mut webview_lock = webview_arc.lock().unwrap();
                if let Some((window, _)) = webview_lock.get_mut(&id) {
                    window.set_title(&title);
                }
            }
            PythonEvent::EvaluateScript(script) => {
                let mut webview_lock = webview_arc.lock().unwrap();
                if let Some((_, webview)) = webview_lock.get_mut(&id) {
                    if let Err(e) = webview.evaluate_script(&script) {
                        eprintln!("Error evaluating script: {:?}", e);
                    }
                }
            }
            _ => {
                eprintln!("Unhandled event: {:?}", event);
            }
        }
    }
}


```
```rust


use std::{collections::HashMap, io};
use connection::Connection;
use serde::{Serialize, Deserialize};
use tao::{
    event::{Event, WindowEvent},
    event_loop::{ControlFlow, EventLoop},
    window::WindowBuilder,
};
use wry::http::Request;
use wry::WebViewBuilder;
use std::sync::{mpsc, Arc, Mutex};

use crate::{
    connection, 
    pythonpipe::PythonCommands, 
    rustpipe::RustCommands, 
    scripts::PYFRAME_WINDOW_SCRIPT, 
    webview_core::tcp_webview_connections, 
    Config
};



pub(crate) fn run_webview(
    config: Config, 
    tcp_done_rx: mpsc::Receiver<()>, 
    connection: Arc<Connection>,
    frompython: Arc<PythonCommands>, 
    fromrust: Arc<RustCommands>
) -> Result<(), Box<dyn std::error::Error>> {
    let mut  webviews = HashMap::new();
    let event_loop = EventLoop::new();
    let window = WindowBuilder::new()
        .with_title(&config.webview.as_ref().unwrap().window_title)
        .build(&event_loop)
        .unwrap();

    #[cfg(any(
        target_os = "windows",
        target_os = "macos",
        target_os = "ios",
        target_os = "android"
    ))]
    let builder = WebViewBuilder::new(&window);

    #[cfg(not(any(
        target_os = "windows",
        target_os = "macos",
        target_os = "ios",
        target_os = "android"
    )))]
    let builder = {
        use tao::platform::unix::WindowExtUnix;
        use wry::WebViewBuilderExtUnix;
        let vbox = window.default_vbox().unwrap();
        WebViewBuilder::new_gtk(vbox)
    };


    let cloned_connection = connection.clone();
    let webview = builder
    //.with_url(&config.webview.as_ref().unwrap().url)
    .with_html(&config.webview.as_ref().unwrap().url)
    .with_devtools(true)
    .with_ipc_handler({

        move |req: Request<String>| {
            if let Some(body_str) = req.body().into() {  
                if body_str.starts_with("#PYFRAME_RESULT:") {  
                    
                    let message = body_str[14..].to_string(); 
                    let connection = cloned_connection.clone();
                    tokio::spawn(async move {
                        let json_value = serde_json::json!({ "data": message });
                        let json_string = json_value.to_string(); // Serialize the JSON object to a string
                        if let Err(e) = connection.emit("message", &json_string).await {
                            eprintln!("Failed to emit message: {:?}", e);
                        }
                    });
                }
            }
        }
    })
    .with_initialization_script(PYFRAME_WINDOW_SCRIPT)
    .build()?;

   
    webviews.insert(window.id(), (window,webview));

    let window = Arc::new(Mutex::new(window));
    

    event_loop.run(move |event, _, control_flow| {
        *control_flow = ControlFlow::Wait;
    
        if let Event::WindowEvent {
            event: WindowEvent::CloseRequested,
            ..
        } = event
        {
            *control_flow = ControlFlow::Exit;
        }
        
        // Clone the necessary variables for the async block
       
        let window = window.clone();
        let frompython = frompython.clone();
        let fromrust = fromrust.clone();
        let win_id = window.clone();
        tokio::spawn(async move {
            
            tcp_webview_connections(frompython, fromrust, &mut webviews, win_id).await;
        });
    
        if tcp_done_rx.try_recv().is_ok() {
            println!("TCP operation completed, closing WebView");
            *control_flow = ControlFlow::Exit;
        }
    });
}





```
