# FFMPEG

# Resize videos

On Windows, put this script in a `.bat`/`.cmd` file.

Usage:

* `ffmpeg` has to be in `PATH`

* Drag and drop a file or a folder on the bat file.

* A `downscaled` folder will be created containing generated videos

* Videos already at or below the target resolution are not processed

* You can override height and fps using CLI (defaut values is 480p 30fps):

```batch
video_downscaler.bat my_video_path height=720 fps=24
```

```batch
@echo off
setlocal enabledelayedexpansion

set "input_path=%~1"
set "downscaled_dir=downscaled"
set "max_height=480"  REM Default height
set "target_fps=30"   REM Default fps

rem Process command-line arguments
:parse_args
if "%~2"=="" goto :done_args

set "arg=%~1"
if "!arg:~0,7!"=="height=" (
    set "max_height=!arg:~7!"
) else if "!arg:~0,4!"=="fps=" (
    set "target_fps=!arg:~4!"
) else (
    echo Invalid argument: %arg%
    goto :usage
)

shift /2
goto :parse_args

:done_args

rem Check if the path exists
if not exist "%input_path%" (
    echo The specified path does not exist.
    exit /b 1
)

rem If the path is a file, extract the folder path
if not exist "%input_path%\*" (
    set "input_file=%~nx1"
    set "input_path=%~dp1"
) else (
    set "input_file="
)

rem Create the "downscaled" directory if it doesn't exist
if not exist "%input_path%\%downscaled_dir%\" (
    mkdir "%input_path%\%downscaled_dir%"
)

rem List of supported video extensions
set "supported_extensions=.mp4 .avi .mkv .mov .wmv"

rem Loop through all video files in the input folder
if "%input_file%"=="" (
    for %%f in ("%input_path%\*.*") do (
        rem Check if the file has a supported extension
        set "ext=%%~xf"
        echo !supported_extensions! |findstr !ext! > nul
        if !errorlevel! == 0 (
            echo Processing: "%%~nxf"
            rem Get the height of the source video using ffprobe
            for /f "delims=" %%a in ('ffprobe -v error -show_entries stream^=height -of default^=noprint_wrappers^=1 "%%~f"') do set "video_height=%%a"
	    for /f %%a in ('ffprobe -v error -select_streams v:0 -show_entries stream^=bit_rate -of default^=noprint_wrappers^=1:nokey^=1 "%%~f"') do (	
   	    	set "input_bitrate=%%a"
	    )

	    if !video_height:~7! gtr %max_height% (
                ffmpeg -init_hw_device cuda=gpu:0 -hwaccel cuvid -hwaccel_output_format cuda -i "%%~f" -c:v h264_nvenc -maxrate !input_bitrate! -bufsize 1M  -n -vf "scale_cuda=w=-1:%max_height%:0:format=yuv420p:force_original_aspect_ratio=decrease,hwdownload,format=yuv420p,fps=%target_fps%,pad=ceil(iw/2)*2:ceil(ih/2)*2,hwupload"  -loglevel quiet -stats "%input_path%\%downscaled_dir%\%%~nf.mp4"
            ) else (
                echo Video is already below or equal to %max_height% pixels. Skipping.
            )
        ) else (
            echo File "%%~nxf" has an unsupported extension. Skipping.
        )
    )
) else (
    rem Process the single input file
    if "%input_path:~-1%"=="\" (
        set "input_path=!input_path:~0,-1!"
    )
    rem Check if the file has a supported extension
    for %%i in ("%input_file%") DO (set ext=%%~xi)
    echo !supported_extensions! |findstr !ext! > nul
    if !errorlevel! == 0 (
        echo Processing: "%input_file%"
        rem Get the height of the source video using ffprobe
        for /f "delims=" %%a in ('ffprobe -v error -show_entries stream^=height -of default^=noprint_wrappers^=1 "%input_path%%input_file%"') do (set "video_height=%%a")
        for /f %%a in ('ffprobe -v error -select_streams v:0 -show_entries stream^=bit_rate -of default^=noprint_wrappers^=1:nokey^=1 "%input_path%%input_file%"') do (	
            set "input_bitrate=%%a"
        )

        if !video_height:~7! gtr !max_height! (
            echo a
            ffmpeg -init_hw_device cuda=gpu:0 -hwaccel cuvid -hwaccel_output_format cuda -i "%input_path%%input_file%" -c:v h264_nvenc -maxrate !input_bitrate! -bufsize 1M -n -vf "scale_cuda=w=-1:%max_height%:0:format=yuv420p:force_original_aspect_ratio=decrease,hwdownload,format=yuv420p,fps=%target_fps%,pad=ceil(iw/2)*2:ceil(ih/2)*2,hwupload" -loglevel quiet -stats  "%input_path%\%downscaled_dir%\%input_file%"
        ) else (
            echo Video is already below or equal to %max_height% pixels. Skipping.
        )
    ) else (
        echo File "%input_file%" has an unsupported extension. Skipping.
    )
)

:usage
echo Usage: your_script.bat "C:\path\to\your\file_or_folder" [height=<desired_height>] [fps=<desired_fps>]
echo Default height: %max_height%
echo Default fps: %target_fps%
echo Example: your_script.bat "C:\path\to\your\file_or_folder" heig	ht=720 fps=24
exit /b 0

echo Done!

```