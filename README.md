# EGLSurfaceView and EGLTextureView
A modified version of GLTextureView designed specifically for use with EGL

```Java
class EGLSurfaceView extends SurfaceView implements SurfaceHolder.Callback
```
```Java
class EGLTextureView extends TextureView implements TextureView.SurfaceTextureListener
```

## Docs

SurfaceView and GLSurfaceView https://source.android.com/devices/graphics/arch-sv-glsv

SurfaceTexture https://source.android.com/devices/graphics/arch-st

TextureView https://source.android.com/devices/graphics/arch-tv

## Unlike TextureView and GLSurfaceView

EGLSurfaceView and EGLTextureView provides dedicated callbacks for EGL context creation and destruction. These dedicated callbacks are required by some rendering api's that need to attach and detach from the EGL context, such as Magnum

EGLSurfaceView and EGLTextureView also syncs its drawing with Choreographer that is provided for each View

This ensures that the view does not overdraw, ensuring smooth rendering

Overdraw means that the buffers are being swapped faster than they are supposed to (no more than once per vsync) resulting in glitches with `random rendering speed boosts` that go beyond 60FPS and then drop to 60FPS

the magic is as follows

```Java
// ...

final Object swapLock = new Object();
boolean shouldSwap;

void swapBuffers() {
    synchronized (swapLock) {
        if (!shouldSwap) {
            shouldSwap = true;
        }
    }
}

Choreographer choreographer = Choreographer.getInstance();

protected void init() {
    // ...
    shouldSwap = false;
    choreographer.postFrameCallback(
            new Choreographer.FrameCallback() {
                @Override
                public void doFrame(long frameTimeNanos) {
                    synchronized (swapLock) {
                        mGLThread.requestRender();
                        choreographer.postFrameCallback(this);
                    }
                }
            }
    );
}

// ...

class GLThread extends Thread {
    // ...
    @Override
    public void run() {
        setName("GLThread " + getId());
        // ...
        try {
            guardedRun();
        } catch (InterruptedException e) {
            // fall through and exit normally
        } finally {
            sGLThreadManager.threadExiting(this);
        }
    }

    private void guardedRun() throws InterruptedException {
        // ...
        try {
            // ...
            while (true) {
                // ...
                synchronized (swapLock) {
                    if (shouldSwap) {
                        int swapError = mEglHelper.swap();
                        shouldSwap = false;
                        // handle errors
                    }
                }
                // ...
            }
        } finally {
            // clean up
        }
    }
}

// ...
```

both EGLSurfaceView and EGLTextureView provide the same API as GLSurfaceView with the following changes

Methods added | Methods depreciated | Methods changed | Information
-|-|-|-
getJavaNameForJNI||| For use with JNI
getJavaSignatureForJNI||| For use with JNI
swapBuffers||| Requests that the render thread swaps its buffers. The buffers will be swapped either when the current draw completes, or when the next draw completes. This may be called from any thread
|||requestRender| calls swapBuffers()
||setRenderMode|| This has no effect on rendering, provided only for compatibility
||getRenderMode|| This has no effect on rendering, provided only for compatibility

both EGLSurfaceView.Renderer and EGLTextureView.Renderer provide the same API as GLSurfaceView.Renderer with the following changes

Methods added | Methods depreciated | Methods changed | Information
-|-|-|-
|||onSurfaceCreated| Not required to override, but can still override if desired
|||onSurfaceDestroyed| Not required to override, but can still override if desired
|||onDrawFrame| you must call swapBuffers() to draw anything
onEglSetup||| Called when the EGL context is made current. Not required to override, but can still override if desired
onEglTearDown||| Called when the EGL context is lost. Not required to override, but can still override if desired
