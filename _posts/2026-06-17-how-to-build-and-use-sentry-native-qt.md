---
title: "How to Build and Integrate sentry-native into a Qt Project"
date:   2026-06-17 10:00:00 +0300
categories:
  - blog
tags:
  - sentry
  - crashes
  - sentry-native
  - qt
  - cmake
---

The Sentry Native SDK is a crash and error reporting client for C and C++ applications. It supports tags, breadcrumbs, and custom context to enrich crash reports - making it easier to reproduce and fix issues in production.
This guide covers building sentry-native with Visual Studio 2022 on Windows and integrating it into a Qt/CMake project, including debug symbol upload for readable stack traces.

## Build sentry-native with VS 2022 Community x64
 
Clone source code with submodules:
```
git clone --recurse-submodules git@github.com:getsentry/sentry-native.git
```

```
cd sentry-native
```
Set up Visual Studio environment:

```
"C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat"
```

Сonfigure the project:

```
cmake -G "Visual Studio 17 2022" -A x64 -B build-x64-release -DSENTRY_BUILD_RUNTIMESTATIC=OFF -DSENTRY_BUILD_SHARED_LIBS=ON -DSENTRY_BACKEND=crashpad -DSENTRY_TRANSPORT=winhttp -DCMAKE_BUILD_TYPE=release -DSENTRY_BUILD_TESTS=OFF -DSENTRY_BUILD_EXAMPLES=OFF
```

build the project:
```
cmake --build build-x64-release --config release --parallel
```

install:
```
cmake --install build-x64-release --config release --prefix C:/sentry-install/x64/release
```

## Use Sentry in a Qt project

Example project for use sentry in Qt/CMake: <https://github.com/danilevsky/sentry-native-qt-demo>

Copy DSN from Sentry: Settings -> Project -> [Your Project] -> Client Keys (DSN)->DSN

```
#include "mainview.h"

#include <QApplication>
#include <sentry.h>

int main(int argc, char *argv[])
{

	std::string sDSN = <SENTRY_DSN_VALUE>;

	sentry_options_t *options = sentry_options_new();
	sentry_options_set_dsn(options, sDSN.c_str());
	sentry_options_set_database_path(options, ".sentry-native");
	sentry_options_set_release(options, "test-app@1.0.1");
	sentry_options_set_debug(options, 1);
	sentry_init(options);

	auto sentryClose = qScopeGuard([] { sentry_close(); });

	QApplication a(argc, argv);
	MainView w;
	w.show();

	sentry_capture_event(sentry_value_new_message_event(
		/*   level */ SENTRY_LEVEL_INFO,
		/*  logger */ "custom",
		/* message */ "It works!"
		));

	return QCoreApplication::exec();
}
```

Upload .pdb and .exe files for crash report analysis with tool sentry-cli. 

The sentry-cli requires an Organization Token so that Debug Information Files can be uploaded.

- Settings->Integrations->Create New Integration->Internal Integration
- In section 'Token' create new Token for SENTRY_AUTH_TOKEN

Set variables and upload files:
```
SET SENTRY_END_POINT=<your_sentry_url>
SET SENTRY_AUTH_TOKEN=<token>
SET SENTRY_ORG=<org>
SET SENTRY_LOG_LEVEL=warn
SET SENTRY_PROJECT_NAME=<project_name>

sentry-cli --url %SENTRY_END_POINT% --auth-token %SENTRY_AUTH_TOKEN% debug-files upload -o %SENTRY_ORG% -p %SENTRY_PROJECT_NAME% "test_app.pdb" --log-level %SENTRY_LOG_LEVEL% --include-sources

sentry-cli --url %SENTRY_END_POINT% --auth-token %SENTRY_AUTH_TOKEN% debug-files upload -o %SENTRY_ORG% -p %SENTRY_PROJECT_NAME% "test_app.exe" --log-level %SENTRY_LOG_LEVEL%
```

Invoke simple crash in app:
```
void MainView::OnClick()
{
	int *p = nullptr;
	*p = 42;
}
```

Check the project dashboard, you will see issues:
 - 'It Works!'
 - Exception with stack trace and minidump in Attachments
 
Links:
 - [Script for build sentry-native for all configurations](https://github.com/danilevsky/sentry-native-build-script)
 - <https://docs.sentry.io/platforms/native/>
 - <https://docs.sentry.io/cli/dif/>
 - <https://github.com/getsentry/sentry-native>
 
