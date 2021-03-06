---
layout: post
title: 毕设记录之Android照相机基础基于camera2API
subtitle: Introduction of Android API Camera2
keyword: Android API Camera2
tag:
   - Android
   - 毕业设计
---
**本文是作者原创文章，欢迎转载，请注明出处 from:@[Eric_Lai](http://laihaotao.github.io)**


# 前言
最近，在使用Android做一个照相机的开发。因为不能使用系统提供的相机应用，所以只能自己写一个。Android以前提供相机的api叫camera，不过在level 21被Google抛弃了。网上的教程，还有很多都是使用camera的，为了好好学习一下camera2，就去扒了Google提供的官方示例。下面给一个github的连接，可以找到全部的源码。

>[Camera2Basic Sample](https://github.com/googlesamples/android-Camera2Basic)

# 源码分析
下述的例子主要提供的功能是：相机预览、拍照、照片保存。具体如下：进入该应用后，可以看到相机的预览画面，提供一个按钮用于拍照，拍完照片后照片会保存在SD卡的根目录下。

## 架构分析
首先来了解一下camera2的整体结构：

![图1 camera2整体架构图](/images/android-camera2-1.jpg)
如上所示，整个camera2由一个CameraManager来进行统一管理，通过Context的getSystemService方法可以实例化CameraManager，然后该类主要通过三个类来对Camera进行操作。下面分别介绍一下：

- CameraDevice：描述一个照相机设备，一个Android设备可能会有多个摄像头，通过CameraId可以进行区别。它最主要有一个相机状态的回调函数，当下达打开相机的命令后，若相机正确的打开便会回调该函数。
- CameraCharacteristic：某个照相机设备的具体参数。本例主要用到它提供的输出格式（即输出数据的格式）。
- CameraCaptureSession：相机捕获会话，通过这个类可以和相机进行对话（预览还是单张拍照还是录像等）。这里有两个回调函数，捕获状态的回调，和捕获数据的回调（后文会有详述）。

上图左上部分所示的时Android设备和camera设备的通信情况，两者之间通过pipeline（管道）进行数据交换。当需要尽心不同的操作时，将CameraCaptureRequest通过管道传给camera，接收到请求后，camera做出相应的反应，将获取到得数据CmaeraMetadata通过管道传回给Android设备。

## 注意事项
本例，需要使用比较多的权限，请参看源码AndroidManifest.xml。

Android设备的屏幕方向，与摄像头的原始方向并不一致，需要做方向转换。一般而言，当Android设备横着放时，与摄像头的方向是一致的。

为了避免照片失真（照片被拉长或者压扁），需要保证预览的长宽比例、照片的长宽比例和相机输出格式的长宽比例三者保持一致。

## 代码流转
本例当中，用一个activity承载一个fragment。所有的代码都写在fragment里面，重写了fragment的几个生命周期函数:

- onCreateView：加载fragment的布局文件；
- onViewCreated：实例化布局控件；
- onActivityCreated：在SD卡的目录下建立jpg文件等待待将拍到的照片写进去；
- onResume：开始照相机线程，执行一些逻辑判断；
- onPause：关闭照相机，停止照相机线程；


![图2 代码整体流程](/images/android-camera2-2.jpg)

正常来说，代码的整体流程如图2所示，activity将需要的fragment加载进来后，开始加载显示预览的控件texture，当控件加载完毕会执行一个回调函数`onSurfaceTextureAvailable()`，在这个回调函数里面，打开摄像头（即执行`openCamera()`）。

`openCamera()`里面，首先要配置相机的输出，预览图像和拍照的图片要作不同的处理，然后根据当前的设备屏幕环境，判断是否需要进行数据的转换，最后通过cameraManager打开摄像头（调用`cameraManager.openCamera()`方法）。

更详细的方法请看后面的源码，图2当中，中间是判断屏幕方向的逻辑，右边是自动选择最合适的显示逻辑。

## 源码

```
// 代码比较长，请耐心查看，注释可能有不正确的地方，请提出
// 注意，下面代码为了配合我的使用，已经去掉了按钮，但是拍照的方法仍然保留，通过调用方法可以完成拍照
package com.eric_lai.weeding_robot.fragment;

/**
 * Created by ERIC_LAI on 16/3/18.
 */

import android.app.Activity;
import android.app.AlertDialog;
import android.app.Dialog;
import android.app.DialogFragment;
import android.app.Fragment;
import android.content.Context;
import android.content.DialogInterface;
import android.content.res.Configuration;
import android.graphics.ImageFormat;
import android.graphics.Matrix;
import android.graphics.Point;
import android.graphics.RectF;
import android.graphics.SurfaceTexture;
import android.hardware.camera2.CameraAccessException;
import android.hardware.camera2.CameraCaptureSession;
import android.hardware.camera2.CameraCharacteristics;
import android.hardware.camera2.CameraDevice;
import android.hardware.camera2.CameraManager;
import android.hardware.camera2.CameraMetadata;
import android.hardware.camera2.CaptureRequest;
import android.hardware.camera2.CaptureResult;
import android.hardware.camera2.TotalCaptureResult;
import android.hardware.camera2.params.StreamConfigurationMap;
import android.media.Image;
import android.media.ImageReader;
import android.os.Bundle;
import android.os.Handler;
import android.os.HandlerThread;
import android.support.annotation.NonNull;
import android.util.Log;
import android.util.Size;
import android.util.SparseIntArray;
import android.view.LayoutInflater;
import android.view.Surface;
import android.view.TextureView;
import android.view.View;
import android.view.ViewGroup;
import android.widget.Toast;

import com.eric_lai.weeding_robot.R;
import com.eric_lai.weeding_robot.view.AutoFitTextureView;

import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.nio.ByteBuffer;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.Comparator;
import java.util.List;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class CameraFragment extends Fragment {

    /**
     * Conversion from screen rotation to JPEG orientation.
     */
    private static final SparseIntArray ORIENTATIONS = new SparseIntArray();
    private static final String FRAGMENT_DIALOG = "dialog";

    static {
        ORIENTATIONS.append(Surface.ROTATION_0, 90);
        ORIENTATIONS.append(Surface.ROTATION_90, 0);
        ORIENTATIONS.append(Surface.ROTATION_180, 270);
        ORIENTATIONS.append(Surface.ROTATION_270, 180);
    }

    /**
     * 调试用TAG
     */
    private static final String TAG = "CameraFragment";

    /**
     * 相机状态:
     * 0: 预览
     * 1: 等待上锁(拍照片前将预览锁上保证图像不在变化)
     * 2: 等待预拍照(对焦, 曝光等操作)
     * 3: 等待非预拍照(闪光灯等操作)
     * 4: 已经获取照片
     */
    private static final int STATE_PREVIEW = 0;
    private static final int STATE_WAITING_LOCK = 1;
    private static final int STATE_WAITING_PRECAPTURE = 2;
    private static final int STATE_WAITING_NON_PRECAPTURE = 3;
    private static final int STATE_PICTURE_TAKEN = 4;

    /**
     * Camera2 API提供的最大预览宽度和高度
     */
    private static final int MAX_PREVIEW_WIDTH = 1920;
    private static final int MAX_PREVIEW_HEIGHT = 1080;

    /**
     * SurfaceTexture监听器
     */
    private final TextureView.SurfaceTextureListener mSurfaceTextureListener
            = new TextureView.SurfaceTextureListener() {

        @Override
        public void onSurfaceTextureAvailable(SurfaceTexture texture, int width, int height) {
            // SurfaceTexture就绪后回调执行打开相机操作
            openCamera(width, height);
        }

        @Override
        public void onSurfaceTextureSizeChanged(SurfaceTexture texture, int width, int height) {
            // 预览方向改变时, 执行转换操作
            configureTransform(width, height);
        }

        @Override
        public boolean onSurfaceTextureDestroyed(SurfaceTexture texture) {
            return true;
        }

        @Override
        public void onSurfaceTextureUpdated(SurfaceTexture texture) {
        }

    };

    /**
     * 正在使用的相机id
     */
    private String mCameraId;

    /**
     * 预览使用的自定义TextureView控件
     */
    private AutoFitTextureView mTextureView;

    /**
     * 预览用的获取会话
     */
    private CameraCaptureSession mCaptureSession;

    /**
     * 正在使用的相机
     */
    private CameraDevice mCameraDevice;

    /**
     * 预览数据的尺寸
     */
    private Size mPreviewSize;

    /**
     * 相机状态改变的回调函数
     */
    private final CameraDevice.StateCallback mStateCallback = new CameraDevice.StateCallback() {

        @Override
        public void onOpened(@NonNull CameraDevice cameraDevice) {
            // 当相机打开执行以下操作:
            // 1. 释放访问许可
            // 2. 将正在使用的相机指向将打开的相机
            // 3. 创建相机预览会话
            mCameraOpenCloseLock.release();
            mCameraDevice = cameraDevice;
            createCameraPreviewSession();
        }

        @Override
        public void onDisconnected(@NonNull CameraDevice cameraDevice) {
            // 当相机失去连接时执行以下操作:
            // 1. 释放访问许可
            // 2. 关闭相机
            // 3. 将正在使用的相机指向null
            mCameraOpenCloseLock.release();
            cameraDevice.close();
            mCameraDevice = null;
        }

        @Override
        public void onError(@NonNull CameraDevice cameraDevice, int error) {
            // 当相机发生错误时执行以下操作:
            // 1. 释放访问许可
            // 2. 关闭相机
            // 3, 将正在使用的相机指向null
            // 4. 获取当前的活动, 并结束它
            mCameraOpenCloseLock.release();
            cameraDevice.close();
            mCameraDevice = null;
            Activity activity = getActivity();
            if (null != activity) {
                activity.finish();
            }
        }

    };

    /**
     * 处理拍照等工作的子线程
     */
    private HandlerThread mBackgroundThread;

    /**
     * 上面定义的子线程的处理器
     */
    private Handler mBackgroundHandler;

    /**
     * 静止页面捕获(拍照)处理器
     */
    private ImageReader mImageReader;

    /**
     * 输出照片的文件
     */
    private File mFile;

    /**
     * ImageReader的回调函数, 其中的onImageAvailable会在照片准备好可以被保存时调用
     */
    private final ImageReader.OnImageAvailableListener mOnImageAvailableListener
            = new ImageReader.OnImageAvailableListener() {

        @Override
        public void onImageAvailable(ImageReader reader) {
            mBackgroundHandler.post(new ImageSaver(reader.acquireNextImage(), mFile));
        }

    };

    /**
     * 预览请求构建器, 用来构建"预览请求"(下面定义的)通过pipeline发送到Camera device
     */
    private CaptureRequest.Builder mPreviewRequestBuilder;

    /**
     * 预览请求, 由上面的构建器构建出来
     */
    private CaptureRequest mPreviewRequest;

    /**
     * 当前的相机状态, 这里初始化为预览, 因为刚载入这个fragment时应显示预览
     */
    private int mState = STATE_PREVIEW;

    /**
     * 信号量控制器, 防止相机没有关闭时退出本应用(若没有关闭就退出, 会造成其他应用无法调用相机)
     * 当某处获得这个许可时, 其他需要许可才能执行的代码需要等待许可被释放才能获取
     */
    private Semaphore mCameraOpenCloseLock = new Semaphore(1);

    /**
     * 捕获会话回调函数
     *
     */
    private CameraCaptureSession.CaptureCallback mCaptureCallback
            = new CameraCaptureSession.CaptureCallback() {

        @Override
        public void onCaptureProgressed(@NonNull CameraCaptureSession session,
                                        @NonNull CaptureRequest request,
                                        @NonNull CaptureResult partialResult) {
            process(partialResult);
        }

        @Override
        public void onCaptureCompleted(@NonNull CameraCaptureSession session,
                                       @NonNull CaptureRequest request,
                                       @NonNull TotalCaptureResult result) {
            process(result);
        }

        // 自定义的一个处理方法
        private void process(CaptureResult result) {
            switch (mState) {
                case STATE_PREVIEW: {
                    // 状态是预览时, 不需要做任何事情
                    break;
                }
                case STATE_WAITING_LOCK: {
                    // 等待锁定的状态, 某些设备完成锁定后CONTROL_AF_STATE可能为null
                    Integer afState = result.get(CaptureResult.CONTROL_AF_STATE);
                    if (afState == null) {
                        captureStillPicture();
                    } else if (CaptureResult.CONTROL_AF_STATE_FOCUSED_LOCKED == afState ||
                            CaptureResult.CONTROL_AF_STATE_NOT_FOCUSED_LOCKED == afState) {
                        // 如果焦点已经锁定(不管自动对焦是否成功), 检查AE的返回, 注意某些设备CONTROL_AE_STATE可
                        // 能为空
                        Integer aeState = result.get(CaptureResult.CONTROL_AE_STATE);
                        if (aeState == null ||
                                aeState == CaptureResult.CONTROL_AE_STATE_CONVERGED) {
                            // 如果自动曝光(AE)设定良好, 将状态置为已经拍照, 执行拍照
                            mState = STATE_PICTURE_TAKEN;
                            captureStillPicture();
                        } else {
                            // 以上条件都不满足, 执行预拍照系列操作
                            runPrecaptureSequence();
                        }
                    }
                    break;
                }
                case STATE_WAITING_PRECAPTURE: {
                    // 等待预处理状态, 某些设备CONTROL_AE_STATE可能为null
                    Integer aeState = result.get(CaptureResult.CONTROL_AE_STATE);
                    if (aeState == null ||
                            aeState == CaptureResult.CONTROL_AE_STATE_PRECAPTURE ||
                            aeState == CaptureRequest.CONTROL_AE_STATE_FLASH_REQUIRED) {
                        // 如果AE需要做于拍照或者需要闪光灯, 将状态置为"非等待预拍照"(翻译得有点勉强)
                        mState = STATE_WAITING_NON_PRECAPTURE;
                    }
                    break;
                }
                case STATE_WAITING_NON_PRECAPTURE: {
                    // 某些设备CONTROL_AE_STATE可能为null
                    Integer aeState = result.get(CaptureResult.CONTROL_AE_STATE);
                    if (aeState == null || aeState != CaptureResult.CONTROL_AE_STATE_PRECAPTURE) {
                        // 如果AE做完"非等待预拍照", 将状态置为已经拍照, 并执行拍照操作
                        mState = STATE_PICTURE_TAKEN;
                        captureStillPicture();
                    }
                    break;
                }
            }
        }
    };

    /**
     * 在UI上显示Toast的方法
     */
    private void showToast(final String text) {
        final Activity activity = getActivity();
        if (activity != null) {
            activity.runOnUiThread(new Runnable() {
                @Override
                public void run() {
                    Toast.makeText(activity, text, Toast.LENGTH_SHORT).show();
                }
            });
        }
    }

    /**
     * 返回最合适的预览尺寸
     *
     * @param choices           相机希望输出类支持的尺寸list
     * @param textureViewWidth  texture view 宽度
     * @param textureViewHeight texture view 高度
     * @param maxWidth          能够选择的最大宽度
     * @param maxHeight         能够选择的醉倒高度
     * @param aspectRatio       图像的比例(pictureSize, 只有当pictureSize和textureSize保持一致, 才不会失真)
     * @return 最合适的预览尺寸
     */
    private static Size chooseOptimalSize(Size[] choices, int textureViewWidth,
                                          int textureViewHeight, int maxWidth, int maxHeight, Size aspectRatio) {

        // 存放小于等于限定尺寸, 大于等于texture控件尺寸的Size
        List<Size> bigEnough = new ArrayList<>();
        // 存放小于限定尺寸, 小于texture控件尺寸的Size
        List<Size> notBigEnough = new ArrayList<>();
        int w = aspectRatio.getWidth();
        int h = aspectRatio.getHeight();
        for (Size option : choices) {
            if (option.getWidth() <= maxWidth && option.getHeight() <= maxHeight &&
                    option.getHeight() == option.getWidth() * h / w) {
                // option.getHeight() == option.getWidth() * h / w 用来保证
                // pictureSize的 w / h 和 textureSize的 w / h 一致
                if (option.getWidth() >= textureViewWidth &&
                        option.getHeight() >= textureViewHeight) {
                    bigEnough.add(option);
                } else {
                    notBigEnough.add(option);
                }
            }
        }

        // 1. 若存在bigEnough数据, 则返回最大里面最小的
        // 2. 若不存bigEnough数据, 但是存在notBigEnough数据, 则返回在最小里面最大的
        // 3. 上述两种数据都没有时, 返回空, 并在日志上显示错误信息
        if (bigEnough.size() > 0) {
            return Collections.min(bigEnough, new CompareSizesByArea());
        } else if (notBigEnough.size() > 0) {
            return Collections.max(notBigEnough, new CompareSizesByArea());
        } else {
            Log.e(TAG, "Couldn't find any suitable preview size");
            return choices[0];
        }
    }

    public static CameraFragment newInstance() {
        return new CameraFragment();
    }

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        return inflater.inflate(R.layout.fragment_camera, container, false);
    }

    @Override
    public void onViewCreated(final View view, Bundle savedInstanceState) {
        mTextureView = (AutoFitTextureView) view.findViewById(R.id.texture);
    }

    @Override
    public void onActivityCreated(Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        mFile = new File(getActivity().getExternalFilesDir(null), "pic.jpg");
    }

    @Override
    public void onResume() {
        super.onResume();
        startBackgroundThread();
        // 当屏幕关闭后重新打开, 若SurfaceTexture已经就绪, 此时onSurfaceTextureAvailable不会被回调, 这种情况下
        // 如果SurfaceTexture已经就绪, 则直接打开相机, 否则等待SurfaceTexture已经就绪的回调
        if (mTextureView.isAvailable()) {
            openCamera(mTextureView.getWidth(), mTextureView.getHeight());
        } else {
            mTextureView.setSurfaceTextureListener(mSurfaceTextureListener);
        }
    }

    @Override
    public void onPause() {
        closeCamera();
        stopBackgroundThread();
        super.onPause();
    }

    /**
     * 设置相机的输出, 包括预览和拍照
     *
     * 处理流程如下:
     * 1. 获取当前的摄像头, 并将拍照输出设置为最高画质
     * 2. 判断显示方向和摄像头传感器方向是否一致, 是否需要旋转画面
     * 3. 获取当前显示尺寸和相机的输出尺寸, 选择最合适的预览尺寸
     *
     * @param width  预览宽度
     * @param height 预览高度
     */
    private void setUpCameraOutputs(int width, int height) {
        // 获取当前活动
        Activity activity = getActivity();
        // 获取CameraManager实例
        CameraManager manager = (CameraManager) activity.getSystemService(Context.CAMERA_SERVICE);
        try {
            // 遍历运行本应用的设备的所有摄像头
            for (String cameraId : manager.getCameraIdList()) {
                CameraCharacteristics characteristics
                        = manager.getCameraCharacteristics(cameraId);

                // 如果该摄像头是前置摄像头, 则看下一个摄像头(本应用不使用前置摄像头)
                Integer facing = characteristics.get(CameraCharacteristics.LENS_FACING);
                if (facing != null && facing == CameraCharacteristics.LENS_FACING_FRONT) {
                    continue;
                }

                StreamConfigurationMap map = characteristics.get(
                        CameraCharacteristics.SCALER_STREAM_CONFIGURATION_MAP);
                if (map == null) {
                    continue;
                }

                // 选用最高画质
                // maxImages是ImageReader一次可以访问的最大图片数量
                Size largest = Collections.max(
                        Arrays.asList(map.getOutputSizes(ImageFormat.JPEG)),
                        new CompareSizesByArea());
                Log.d(TAG, "largest.width: " + largest.getWidth());
                Log.d(TAG, "largest.height: " + largest.getHeight());
                mImageReader = ImageReader.newInstance(largest.getWidth(), largest.getHeight(),
                        ImageFormat.JPEG, /*maxImages*/2);
                mImageReader.setOnImageAvailableListener(
                        mOnImageAvailableListener, mBackgroundHandler);

                // 获取手机目前的旋转方向(横屏还是竖屏, 对于"自然"状态下高度大于宽度的设备来说横屏是ROTATION_90
                // 或者ROTATION_270,竖屏是ROTATION_0或者ROTATION_180)
                int displayRotation = activity.getWindowManager().getDefaultDisplay().getRotation();
                // 获取相机传感器的方向("自然"状态下垂直放置为0, 顺时针算起, 每次加90读)
                // 注意, 这个参数, 是由设备的生产商来决定的, 大多数情况下, 该值为90, 以下的switch这么写
                // 是为了配适某些特殊的手机
                int sensorOrientation =
                        characteristics.get(CameraCharacteristics.SENSOR_ORIENTATION);
                boolean swappedDimensions = false;
                Log.d(TAG, "displayRotation: " + displayRotation);
                Log.d(TAG, "sensorOritentation: " + sensorOrientation);
                switch (displayRotation) {
                    // ROTATION_0和ROTATION_180都是竖屏只需做同样的处理操作
                    // 显示为竖屏时, 若传感器方向为90或者270, 则需要进行转换(标志位置true)
                    case Surface.ROTATION_0:
                    case Surface.ROTATION_180:
                        if (sensorOrientation == 90 || sensorOrientation == 270) {
                            Log.d(TAG, "swappedDimensions set true !");
                            swappedDimensions = true;
                        }
                        break;
                    // ROTATION_90和ROTATION_270都是横屏只需做同样的处理操作
                    // 显示为横屏时, 若传感器方向为0或者180, 则需要进行转换(标志位置true)
                    case Surface.ROTATION_90:
                    case Surface.ROTATION_270:
                        if (sensorOrientation == 0 || sensorOrientation == 180) {
                            swappedDimensions = true;
                        }
                        break;
                    default:
                        Log.e(TAG, "Display rotation is invalid: " + displayRotation);
                }

                // 获取当前的屏幕尺寸, 放到一个点对象里
                Point displaySize = new Point();
                activity.getWindowManager().getDefaultDisplay().getSize(displaySize);
                // 旋转前的预览宽度(相机给出的), 通过传进来的参数获得
                int rotatedPreviewWidth = width;
                // 旋转前的预览高度(相机给出的), 通过传进来的参数获得
                int rotatedPreviewHeight = height;
                // 将当前的显示尺寸赋给最大的预览尺寸(能够显示的尺寸, 用来计算用的(texture可能比它小需要配适))
                int maxPreviewWidth = displaySize.x;
                int maxPreviewHeight = displaySize.y;

                // 如果需要进行画面旋转, 将宽度和高度对调
                if (swappedDimensions) {
                    rotatedPreviewWidth = height;
                    rotatedPreviewHeight = width;
                    maxPreviewWidth = displaySize.y;
                    maxPreviewHeight = displaySize.x;
                }

                // 尺寸太大时的极端处理
                if (maxPreviewWidth > MAX_PREVIEW_WIDTH) {
                    maxPreviewWidth = MAX_PREVIEW_WIDTH;
                }

                if (maxPreviewHeight > MAX_PREVIEW_HEIGHT) {
                    maxPreviewHeight = MAX_PREVIEW_HEIGHT;
                }

                // 自动计算出最适合的预览尺寸
                // 第一个参数:map.getOutputSizes(SurfaceTexture.class)表示SurfaceTexture支持的尺寸List
                mPreviewSize = chooseOptimalSize(map.getOutputSizes(SurfaceTexture.class),
                        rotatedPreviewWidth, rotatedPreviewHeight, maxPreviewWidth,
                        maxPreviewHeight, largest);

                // 获取当前的屏幕方向
                int orientation = getResources().getConfiguration().orientation;
                if (orientation == Configuration.ORIENTATION_LANDSCAPE) {
                    // 如果方向是横向(landscape)
                    mTextureView.setAspectRatio(
                            mPreviewSize.getWidth(), mPreviewSize.getHeight());
                } else {
                    // 方向不是横向(即竖向)
                    mTextureView.setAspectRatio(
                            mPreviewSize.getHeight(), mPreviewSize.getWidth());
                }

                Log.d(TAG, "real preview width: " + rotatedPreviewWidth);
                Log.d(TAG, "real preview height: " + rotatedPreviewHeight);
//                Log.d(TAG, "max preview width: " + maxPreviewWidth);
//                Log.d(TAG, "max preview width: : " + maxPreviewHeight);
                // 下面这两个是计算后的previewSize=======================================
                Log.d(TAG, "mPreviewSize.getWidth: " + mPreviewSize.getWidth());
                Log.d(TAG, "mPreviewSize.getHeight: " + mPreviewSize.getHeight());
                // =================================================================

                mCameraId = cameraId;
                return;
            }
        } catch (CameraAccessException e) {
            e.printStackTrace();
        } catch (NullPointerException e) {
            // 对话框显示错误
            ErrorDialog.newInstance(getString(R.string.camera_error))
                    .show(getChildFragmentManager(), FRAGMENT_DIALOG);
        }
    }

    /**
     * 通过cameraId打开特定的相机
     */
    private void openCamera(int width, int height) {
        // 设置相机输出
        setUpCameraOutputs(width, height);
        // 配置格式转换
        configureTransform(width, height);
        // 获取当前活动和CameraManager的实例
        Activity activity = getActivity();
        CameraManager manager = (CameraManager) activity.getSystemService(Context.CAMERA_SERVICE);
        try {
            // 尝试获得相机开打关闭许可, 等待2500时间仍没有获得则排除异常
            if (!mCameraOpenCloseLock.tryAcquire(2500, TimeUnit.MILLISECONDS)) {
                throw new RuntimeException("Time out waiting to lock camera opening.");
            }
            // 打开相机, 参数是: 相机id, 相机状态回调, 子线程处理器
            manager.openCamera(mCameraId, mStateCallback, mBackgroundHandler);
        } catch (CameraAccessException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            throw new RuntimeException("Interrupted while trying to lock camera opening.", e);
        }
    }

    /**
     * 关闭正在使用的相机
     */
    private void closeCamera() {
        try {
            // 获得相机开打关闭许可
            mCameraOpenCloseLock.acquire();
            // 关闭捕获会话
            if (null != mCaptureSession) {
                mCaptureSession.close();
                mCaptureSession = null;
            }
            // 关闭当前相机
            if (null != mCameraDevice) {
                mCameraDevice.close();
                mCameraDevice = null;
            }
            // 关闭拍照处理器
            if (null != mImageReader) {
                mImageReader.close();
                mImageReader = null;
            }
        } catch (InterruptedException e) {
            throw new RuntimeException("Interrupted while trying to lock camera closing.", e);
        } finally {
            // 释放相机开打关闭许可
            mCameraOpenCloseLock.release();
        }
    }

    /**
     * 开启子线程
     */
    private void startBackgroundThread() {
        mBackgroundThread = new HandlerThread("CameraBackground");
        mBackgroundThread.start();
        mBackgroundHandler = new Handler(mBackgroundThread.getLooper());
    }

    /**
     * 停止子线程
     */
    private void stopBackgroundThread() {
        mBackgroundThread.quitSafely();
        try {
            mBackgroundThread.join();
            mBackgroundThread = null;
            mBackgroundHandler = null;
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    /**
     * 创建预览对话
     */
    private void createCameraPreviewSession() {
        try {
            // 获取texture实例
            SurfaceTexture texture = mTextureView.getSurfaceTexture();
            assert texture != null;
            // 设置宽度和高度
            texture.setDefaultBufferSize(mPreviewSize.getWidth(), mPreviewSize.getHeight());
            // 用来开始预览的输出surface
            Surface surface = new Surface(texture);
            // 预览请求构建
            mPreviewRequestBuilder
                    = mCameraDevice.createCaptureRequest(CameraDevice.TEMPLATE_PREVIEW);
            mPreviewRequestBuilder.addTarget(surface);
            // 创建预览的捕获会话
            mCameraDevice.createCaptureSession(Arrays.asList(surface, mImageReader.getSurface()),
                    new CameraCaptureSession.StateCallback() {

                        @Override
                        public void onConfigured(@NonNull CameraCaptureSession cameraCaptureSession) {
                            // 相机关闭时, 直接返回
                            if (null == mCameraDevice) {
                                return;
                            }

                            // 会话可行时, 将构建的会话赋给field
                            mCaptureSession = cameraCaptureSession;
                            try {
                                // 自动对焦
                                mPreviewRequestBuilder.set(CaptureRequest.CONTROL_AF_MODE,
                                        CaptureRequest.CONTROL_AF_MODE_CONTINUOUS_PICTURE);
                                // 自动闪光
                                mPreviewRequestBuilder.set(CaptureRequest.CONTROL_AE_MODE,
                                        CaptureRequest.CONTROL_AE_MODE_ON_AUTO_FLASH);

                                // 构建上述的请求
                                mPreviewRequest = mPreviewRequestBuilder.build();
                                // 重复进行上面构建的请求, 以便显示预览
                                mCaptureSession.setRepeatingRequest(mPreviewRequest,
                                        mCaptureCallback, mBackgroundHandler);
                            } catch (CameraAccessException e) {
                                e.printStackTrace();
                            }
                        }

                        @Override
                        public void onConfigureFailed(
                                @NonNull CameraCaptureSession cameraCaptureSession) {
                            showToast("Failed");
                        }
                    }, null
            );
        } catch (CameraAccessException e) {
            e.printStackTrace();
        }
    }

    /**
     * 屏幕方向发生改变时调用转换数据方法
     *
     * @param viewWidth  mTextureView 的宽度
     * @param viewHeight mTextureView 的高度
     */
    private void configureTransform(int viewWidth, int viewHeight) {
        Activity activity = getActivity();
        if (null == mTextureView || null == mPreviewSize || null == activity) {
            return;
        }
        int rotation = activity.getWindowManager().getDefaultDisplay().getRotation();
        Matrix matrix = new Matrix();
        RectF viewRect = new RectF(0, 0, viewWidth, viewHeight);
        RectF bufferRect = new RectF(0, 0, mPreviewSize.getHeight(), mPreviewSize.getWidth());
        float centerX = viewRect.centerX();
        float centerY = viewRect.centerY();
        if (Surface.ROTATION_90 == rotation || Surface.ROTATION_270 == rotation) {
            bufferRect.offset(centerX - bufferRect.centerX(), centerY - bufferRect.centerY());
            matrix.setRectToRect(viewRect, bufferRect, Matrix.ScaleToFit.FILL);
            float scale = Math.max(
                    (float) viewHeight / mPreviewSize.getHeight(),
                    (float) viewWidth / mPreviewSize.getWidth());
            matrix.postScale(scale, scale, centerX, centerY);
            matrix.postRotate(90 * (rotation - 2), centerX, centerY);
        } else if (Surface.ROTATION_180 == rotation) {
            matrix.postRotate(180, centerX, centerY);
        }
        mTextureView.setTransform(matrix);
    }

    /**
     * 实现拍照的方法
     */
    private void takePicture() {
        lockFocus();
    }

    /**
     * 锁定焦点(拍照的第一步)
     */
    private void lockFocus() {
        try {
            // 构建自动对焦请求
            mPreviewRequestBuilder.set(CaptureRequest.CONTROL_AF_TRIGGER,
                    CameraMetadata.CONTROL_AF_TRIGGER_START);
            // 告诉mCaptureCallback回调状态
            mState = STATE_WAITING_LOCK;
            // 提交一个捕获单一图片的请求个相机
            mCaptureSession.capture(mPreviewRequestBuilder.build(), mCaptureCallback,
                    mBackgroundHandler);
        } catch (CameraAccessException e) {
            e.printStackTrace();
        }
    }

    /**
     * 执行预拍照操作
     */
    private void runPrecaptureSequence() {
        try {
            // 构建预拍照请求
            mPreviewRequestBuilder.set(CaptureRequest.CONTROL_AE_PRECAPTURE_TRIGGER,
                    CaptureRequest.CONTROL_AE_PRECAPTURE_TRIGGER_START);
            // 告诉mCaptureCallback回调状态
            mState = STATE_WAITING_PRECAPTURE;
            // 提交一个捕获单一图片的请求个相机
            mCaptureSession.capture(mPreviewRequestBuilder.build(), mCaptureCallback,
                    mBackgroundHandler);
        } catch (CameraAccessException e) {
            e.printStackTrace();
        }
    }

    /**
     * 拍照操作
     */
    private void captureStillPicture() {
        try {
            final Activity activity = getActivity();
            if (null == activity || null == mCameraDevice) {
                return;
            }
            // This is the CaptureRequest.Builder that we use to take a picture.
            final CaptureRequest.Builder captureBuilder =
                    mCameraDevice.createCaptureRequest(CameraDevice.TEMPLATE_STILL_CAPTURE);
            captureBuilder.addTarget(mImageReader.getSurface());

            // Use the same AE and AF modes as the preview.
            captureBuilder.set(CaptureRequest.CONTROL_AF_MODE,
                    CaptureRequest.CONTROL_AF_MODE_CONTINUOUS_PICTURE);
            captureBuilder.set(CaptureRequest.CONTROL_AE_MODE,
                    CaptureRequest.CONTROL_AE_MODE_ON_AUTO_FLASH);

            // Orientation
            int rotation = activity.getWindowManager().getDefaultDisplay().getRotation();
            captureBuilder.set(CaptureRequest.JPEG_ORIENTATION, ORIENTATIONS.get(rotation));

            CameraCaptureSession.CaptureCallback CaptureCallback
                    = new CameraCaptureSession.CaptureCallback() {

                @Override
                public void onCaptureCompleted(@NonNull CameraCaptureSession session,
                                               @NonNull CaptureRequest request,
                                               @NonNull TotalCaptureResult result) {
                    showToast("Saved: " + mFile);
                    Log.d(TAG, mFile.toString());
                    unlockFocus();
                }
            };

            mCaptureSession.stopRepeating();
            mCaptureSession.capture(captureBuilder.build(), CaptureCallback, null);
        } catch (CameraAccessException e) {
            e.printStackTrace();
        }
    }

    /**
     * 解开锁定的焦点
     */
    private void unlockFocus() {
        try {
            // 构建失能AF的请求
            mPreviewRequestBuilder.set(CaptureRequest.CONTROL_AF_TRIGGER,
                    CameraMetadata.CONTROL_AF_TRIGGER_CANCEL);
            // 构建自动闪光请求(之前拍照前会构建为需要或者不需要闪光灯, 这里重新设回自动)
            mPreviewRequestBuilder.set(CaptureRequest.CONTROL_AE_MODE,
                    CaptureRequest.CONTROL_AE_MODE_ON_AUTO_FLASH);
            // 提交以上构建的请求
            mCaptureSession.capture(mPreviewRequestBuilder.build(), mCaptureCallback,
                    mBackgroundHandler);
            // 拍完照后, 设置成预览状态, 并重复预览请求
            mState = STATE_PREVIEW;
            mCaptureSession.setRepeatingRequest(mPreviewRequest, mCaptureCallback,
                    mBackgroundHandler);
        } catch (CameraAccessException e) {
            e.printStackTrace();
        }
    }

    /**
     * 保存jpeg到指定的文件夹下, 开启子线程执行保存操作
     */
    private static class ImageSaver implements Runnable {

        /**
         * jpeg格式的文件
         */
        private final Image mImage;
        /**
         * 保存的文件
         */
        private final File mFile;

        public ImageSaver(Image image, File file) {
            mImage = image;
            mFile = file;
        }

        @Override
        public void run() {
            ByteBuffer buffer = mImage.getPlanes()[0].getBuffer();
            byte[] bytes = new byte[buffer.remaining()];
            buffer.get(bytes);
            FileOutputStream output = null;
            try {
                output = new FileOutputStream(mFile);
                output.write(bytes);
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                mImage.close();
                if (null != output) {
                    try {
                        output.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }
        }

    }

    /**
     * 比较两个Size的大小基于它们的area
     */
    static class CompareSizesByArea implements Comparator<Size> {

        @Override
        public int compare(Size lhs, Size rhs) {
            // We cast here to ensure the multiplications won't overflow
            return Long.signum((long) lhs.getWidth() * lhs.getHeight() -
                    (long) rhs.getWidth() * rhs.getHeight());
        }

    }

    /**
     * 显示错误信息的对话框
     */
    public static class ErrorDialog extends DialogFragment {

        private static final String ARG_MESSAGE = "message";

        public static ErrorDialog newInstance(String message) {
            ErrorDialog dialog = new ErrorDialog();
            Bundle args = new Bundle();
            args.putString(ARG_MESSAGE, message);
            dialog.setArguments(args);
            return dialog;
        }

        @Override
        public Dialog onCreateDialog(Bundle savedInstanceState) {
            final Activity activity = getActivity();
            return new AlertDialog.Builder(activity)
                    .setMessage(getArguments().getString(ARG_MESSAGE))
                    .setPositiveButton(android.R.string.ok, new DialogInterface.OnClickListener() {
                        @Override
                        public void onClick(DialogInterface dialogInterface, int i) {
                            activity.finish();
                        }
                    })
                    .create();
        }

    }

}
```