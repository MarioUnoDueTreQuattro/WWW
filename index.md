---
layout: default
---
## On this site I will put things related to **IT** and what I develop. ##

<br>

### **2025-12-05** ###

I'm currently developing a C++/Qt5 compressed archive manager that uses the bit7z library.

Compressing:

![Compressing](./assets/images/2025-12-05_12-03-31.png)

Compression finished:

![Compression finished](./assets/images/ArchiveManager_2025-12-05_11-58-35.png)

<br>

### **2025-12-06** ###

GUI changes:

![GUI changes](./assets/images/ArchiveManager_2025-12-06_15-03-52.png)

<br>

### **2025-12-08** ###

Main window, under construction:

<img src="./assets/images/ArchiveManager_2025-12-08_07-57-05.png" alt="Main window, under construction" style="max-width: none;">

<br>

### **2025-12-17** ###

Main window, still under construction:

<img src="./assets/images/ArchiveManager_2025-12-17_12-47-11.png" alt="Main window, under construction" style="max-width: none;">

Compression options:

![Compression options](./assets/images/ArchiveManager_2025-12-17_12-49-32.png)

Compression running:

![Compression running](./assets/images/ArchiveManager_2025-12-17_12-50-17.png)

<br>

### **2025-12-18** ###

Polished:

![Compression running](./assets/images/ArchiveManager_2025-12-18_18-49-33.png)

  Buttons are subclassed from QPushButton.

  ```c++
    /**
     * @brief Custom paint event to draw the button elements.
     * @param event The paint event data.
     *
     * This implementation:
     * 1. Draws the native button bevel/background.
     * 2. Calculates the OS-specific "Shift" (e.g., 1px down/right when pressed).
     * 3. Draws the icon fixed to the left (shifted if pressed).
     * 4. Draws the text centered (shifted if pressed).
     * 5. Performs collision detection: if the button is shrunk manually and text
     *    overlaps the icon, the text is pushed to the right.
     */
```

ProgressBar is a QProgressBar subclass without chunks, solid fill.

```c++
#ifndef SOLIDPROGRESSBAR_H
#define SOLIDPROGRESSBAR_H

#include <QProgressBar>

class SolidProgressBar : public QProgressBar
{
    Q_OBJECT
public:
    explicit SolidProgressBar(QWidget *parent = nullptr);
    QSize sizeHint() const override;
    QSize minimumSizeHint() const override;
protected:
    void paintEvent(QPaintEvent *event) override;
};
#endif // SOLIDPROGRESSBAR_H
```

"Show" checkbox select file in any file manager:

```c++
void Helper::openFolderAndSelectFileEx(const QString &filePath)
{
    QFileInfo fi(filePath);
    if (!fi.exists())
    {
        qDebug() << "File does not exist:" << filePath;
        return;
    }
    // Step 1: Resolve file symlink if it exists
    QString targetFilePath = fi.absoluteFilePath();
    if (fi.isSymLink())
    {
        QString realFile = fi.symLinkTarget();
        if (!realFile.isEmpty())
        {
            targetFilePath = realFile;
        }
        else
        {
            qDebug() << "File is a broken symlink:" << filePath;
            targetFilePath = fi.absoluteFilePath(); // fallback: select the symlink itself
        }
    }
    // Step 2: Resolve folder symlink if it exists
    QFileInfo folderInfo(QFileInfo(targetFilePath).absolutePath());
    QString folderPath = folderInfo.absoluteFilePath();
    if (folderInfo.isSymLink())
    {
        QString realFolder = folderInfo.symLinkTarget();
        if (!realFolder.isEmpty())
        {
            folderPath = realFolder;
        }
        else
        {
            qDebug() << "Folder is a broken symlink:" << folderInfo.absoluteFilePath();
            folderPath = folderInfo.absoluteFilePath(); // fallback: open the symlink folder
        }
    }
    // Step 3: Combine resolved folder path and file name
    QString nativePath = QDir::toNativeSeparators(folderPath + "\\" + QFileInfo(targetFilePath).fileName());
    LPCWSTR path = reinterpret_cast<LPCWSTR>(nativePath.utf16());
    // Step 4: Try using the Windows Shell API to select the file
    PIDLIST_ABSOLUTE pidl = nullptr;
    HRESULT hr = SHParseDisplayName(path, nullptr, &pidl, 0, nullptr);
    if (SUCCEEDED(hr) && pidl != nullptr)
    {
        hr = SHOpenFolderAndSelectItems(pidl, 0, nullptr, 0);
        if (FAILED(hr))
        {
            qDebug() << "Shell API failed, falling back to opening folder:" << nativePath;
            QProcess::startDetached("explorer.exe", QStringList() << QDir::toNativeSeparators(folderPath));
        }
        CoTaskMemFree(pidl);
    }
    else
    {
        // Step 5: Shell API failed (e.g., path contains special characters), fallback
        qDebug() << "Failed to parse path with Shell API, opening folder:" << nativePath;
        QProcess::startDetached("explorer.exe", QStringList() << QDir::toNativeSeparators(folderPath));
    }
}
```




* * *
<br>

Code of CompressionProgressFunctor:

```c++
class CompressionProgressFunctor
{
public:
    explicit CompressionProgressFunctor(ArchiveTask* task, uint64_t totalSize, int totalFiles, int zeroByteFiles)
        : m_task(task),
          m_totalSize(totalSize),
          m_totalFiles(totalFiles),
          m_lastPercent(0),
          m_filesProcessed(new int(zeroByteFiles)),  // Pointer to shared counter
          m_startTime(0),
          m_lastUpdateTime(0),
          m_frequency(0),
          m_pausedTime(0),
          m_zeroByteFiles(zeroByteFiles)
    {
        // Get current time in milliseconds (Windows compatible)
        LARGE_INTEGER frequency;
        QueryPerformanceFrequency(&frequency);
        m_frequency = frequency.QuadPart;
        LARGE_INTEGER counter;
        QueryPerformanceCounter(&counter);
        m_startTime = counter.QuadPart;
        m_lastUpdateTime = m_startTime;
    }

    // Copy constructor - share the counter
    CompressionProgressFunctor(const CompressionProgressFunctor& other)
        : m_task(other.m_task),
          m_totalSize(other.m_totalSize),
          m_totalFiles(other.m_totalFiles),
          m_lastPercent(other.m_lastPercent),
          m_filesProcessed(other.m_filesProcessed),  // Share the same pointer
          m_startTime(other.m_startTime),
          m_lastUpdateTime(other.m_lastUpdateTime),
          m_frequency(other.m_frequency),
          m_pausedTime(other.m_pausedTime),
          m_zeroByteFiles(other.m_zeroByteFiles)
    {}

    bool operator()(uint64_t processedSize) const
    {
        if (m_task)
        {
            // Check if paused
            if (m_task->isPaused())
            {
                LARGE_INTEGER pauseStart;
                QueryPerformanceCounter(&pauseStart);
                // Wait while paused
                m_task->waitWhilePaused();
                // Calculate how long we were paused and add to total paused time
                LARGE_INTEGER pauseEnd;
                QueryPerformanceCounter(&pauseEnd);
                m_pausedTime += (pauseEnd.QuadPart - pauseStart.QuadPart);
            }
            // Check if cancelled
            if (m_task->isCancelled())
            {
                return false;  // Stop the operation
            }
        }
        if (m_task && m_totalSize > 0)
        {
            int percent = static_cast<int>((100.0 * processedSize) / m_totalSize);
            if (percent > 100) percent = 100;
            // Get current time
            LARGE_INTEGER counter;
            QueryPerformanceCounter(&counter);
            qint64 currentTime = counter.QuadPart;
            // Calculate elapsed time in seconds
            double elapsedSeconds = static_cast<double>(currentTime - m_startTime) / m_frequency;
            // Calculate speed (MB/s)
            double speedMBps = 0.0;
            if (elapsedSeconds > 0.0)
            {
                speedMBps = (processedSize / (1024.0 * 1024.0)) / elapsedSeconds;
            }
            // Calculate ETA (seconds)
            int etaSeconds = 0;
            if (speedMBps > 0.0 && processedSize > 0)
            {
                qint64 remainingBytes = m_totalSize - processedSize;
                double remainingMB = remainingBytes / (1024.0 * 1024.0);
                etaSeconds = static_cast<int>(remainingMB / speedMBps);
            }
            // Update every 100ms or when percent changes
            double timeSinceLastUpdate = static_cast<double>(currentTime - m_lastUpdateTime) / m_frequency;
            if (percent != m_lastPercent || timeSinceLastUpdate >= 0.1)
            {
                QMetaObject::invokeMethod(m_task->getManager(), "progressUpdated",
                    Qt::QueuedConnection,
                    Q_ARG(int, percent));
                QMetaObject::invokeMethod(m_task->getManager(), "detailedProgressUpdated",
                    Qt::QueuedConnection,
                    Q_ARG(int, percent),
                    Q_ARG(qint64, static_cast<qint64>(processedSize)),
                    Q_ARG(qint64, static_cast<qint64>(m_totalSize)),
                    Q_ARG(int, *m_filesProcessed),  // Dereference pointer
                    Q_ARG(int, m_totalFiles),
                    Q_ARG(double, speedMBps),
                    Q_ARG(int, etaSeconds));
                m_lastPercent = percent;
                m_lastUpdateTime = currentTime;
            }
            return true;  // Continue operation
        }
        // Fallback if no total size
        if (m_task)
        {
            int estimatedPercent = static_cast<int>(processedSize / 102400);
            if (estimatedPercent > 95) estimatedPercent = 95;
            if (estimatedPercent != m_lastPercent)
            {
                m_task->emitProgress(estimatedPercent);
                m_lastPercent = estimatedPercent;
            }
        }
        return m_task ? !m_task->isCancelled() : true;
    }

    void incrementFileCount() const
    {
        if (m_filesProcessed)
        {
            (*m_filesProcessed)++;
        }
    }

    int *getFileCounterPtr() const
    {
        return m_filesProcessed;
    }

private:
    ArchiveTask *m_task;
    uint64_t m_totalSize;
    int m_totalFiles;
    mutable int m_lastPercent;
    int *m_filesProcessed;
    mutable qint64 m_startTime;
    mutable qint64 m_lastUpdateTime;
    qint64 m_frequency;
    mutable qint64 m_pausedTime;
    int m_zeroByteFiles;
};
```

* * *

```
This is the END :-)
```
<dl>
<dt>Name</dt>
<dd>Andrea</dd>
<dt>Born</dt>
<dd>1973</dd>
<dt>Birthplace</dt>
<dd>Italy</dd>
<dt>Color</dt>
<dd>Blue</dd>
</dl>
