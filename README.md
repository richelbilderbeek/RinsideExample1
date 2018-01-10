# RinsideExample1

Rinside example 1

![](CppRinsideExample1.png)

## CppRinsideExample1.pro

```
include(../../DesktopApplication.pri)
include(../../Libraries/Rinside.pri)

HEADERS = qtdensity.h
SOURCES = \
  qtdensity.cpp \
  main.cpp
```


## main.cpp


```c++
// -*- mode: C++; c-indent-level: 4; c-basic-offset: 4;  tab-width: 8; -*-
//
// Qt usage example for RInside, inspired by the standard 'density
// sliders' example for other GUI toolkits
//
// Copyright (C) 2011  Dirk Eddelbuettel and Romain Francois


#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Weffc++"
#pragma GCC diagnostic ignored "-Wunused-local-typedefs"
#pragma GCC diagnostic ignored "-Wunused-but-set-parameter"
#pragma GCC diagnostic ignored "-Wextra"
#include <QApplication>
#include "qtdensity.h"
#pragma GCC diagnostic pop

int main(int argc, char *argv[])
{
  RInside R(argc, argv);   // create an embedded R instance
  QApplication app(argc, argv);
  QtDensity qtdensity(R); // pass R inst. by reference
  return app.exec();
}
```


## qtdensity.h

```c++
// -*- mode: C++; c-indent-level: 4; c-basic-offset: 4; indent-tabs-mode: nil; -*-
//
// Qt usage example for RInside, inspired by the standard 'density
// sliders' example for other GUI toolkits
//
// Copyright (C) 2011 - 2013  Dirk Eddelbuettel and Romain Francois

#ifndef QTDENSITY_H
#define QTDENSITY_H

#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Weffc++"
#pragma GCC diagnostic ignored "-Wunused-local-typedefs"
#pragma GCC diagnostic ignored "-Wunused-but-set-parameter"
#pragma GCC diagnostic ignored "-Wextra"
#include <RInside.h>

#include <QtGui>
#include <QWidget>
#include <QLabel>
#include <QLineEdit>
#include <QRadioButton>
#include <QGroupBox>
#include <QButtonGroup>
#include <QMainWindow>
#include <QHBoxLayout>
#include <QSlider>
#include <QSpinBox>
#include <QLabel>
#include <QTemporaryFile>
#include <QSvgWidget>

class QtDensity : public QMainWindow
{
    Q_OBJECT

    public:
    QtDensity(RInside & R);

private slots:
    void getBandwidth(int bw);
    void getKernel(int kernel);
    void getRandomDataCmd(QString txt);
    void runRandomDataCmd(void);

private:
    void setupDisplay(void);    // standard GUI boilderplate of arranging things
    void plot(void);            // run a density plot in R and update the
    void filterFile(void);      // modify the richer SVG produced by R

    QSvgWidget *m_svg;          // the SVG device
    RInside & m_R;              // reference to the R instance passed to constructor
    QString m_tempfile;         // name of file used by R for plots
    QString m_svgfile;          // another temp file, this time from Qt
    int m_bw, m_kernel;         // parameters used to estimate the density
    QString m_cmd;              // random draw command string
};
#pragma GCC diagnostic pop

#endif
```

## qtdensity.cpp

```c++
// -*- mode: C++; c-indent-level: 4; c-basic-offset: 4; indent-tabs-mode: nil; -*-
//
// Qt usage example for RInside, inspired by the standard 'density
// sliders' example for other GUI toolkits -- this time with SVG
//
// Copyright (C) 2011 - 2013  Dirk Eddelbuettel and Romain Francois

#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Weffc++"
#pragma GCC diagnostic ignored "-Wunused-local-typedefs"
#pragma GCC diagnostic ignored "-Wunused-but-set-parameter"
#pragma GCC diagnostic ignored "-Wextra"
#include "qtdensity.h"

QtDensity::QtDensity(RInside & R) : m_R(R)
{
    m_bw = 100;                 // initial bandwidth, will be scaled by 100 so 1.0
    m_kernel = 0;               // initial kernel: gaussian
    m_cmd = "c(rnorm(100,0,1), rnorm(50,5,1))"; // simple mixture
    m_R["bw"] = m_bw;           // pass bandwidth to R, and have R compute a temp.file name
    m_tempfile = QString::fromStdString(Rcpp::as<std::string>(m_R.parseEval("tfile <- tempfile()")));
    m_svgfile = QString::fromStdString(Rcpp::as<std::string>(m_R.parseEval("sfile <- tempfile()")));
    setupDisplay();
}

void QtDensity::setupDisplay(void)  {
    QWidget *window = new QWidget;
    window->setWindowTitle("Qt and RInside demo: density estimation");

    QSpinBox *spinBox = new QSpinBox;
    QSlider *slider = new QSlider(Qt::Horizontal);
    spinBox->setRange(5, 200);
    slider->setRange(5, 200);
    QObject::connect(spinBox, SIGNAL(valueChanged(int)), slider, SLOT(setValue(int)));
    QObject::connect(slider, SIGNAL(valueChanged(int)), spinBox, SLOT(setValue(int)));
    spinBox->setValue(m_bw);
    QObject::connect(spinBox, SIGNAL(valueChanged(int)), this, SLOT(getBandwidth(int)));

    QLabel *cmdLabel = new QLabel("R command for random data creation");
    QLineEdit *cmdEntry = new QLineEdit(m_cmd);
    QObject::connect(cmdEntry,  SIGNAL(textEdited(QString)), this, SLOT(getRandomDataCmd(QString)));
    QObject::connect(cmdEntry,  SIGNAL(editingFinished()), this, SLOT(runRandomDataCmd()));

    QGroupBox *kernelRadioBox = new QGroupBox("Density Estimation kernel");
    QRadioButton *radio1 = new QRadioButton("&Gaussian");
    QRadioButton *radio2 = new QRadioButton("&Epanechnikov");
    QRadioButton *radio3 = new QRadioButton("&Rectangular");
    QRadioButton *radio4 = new QRadioButton("&Triangular");
    QRadioButton *radio5 = new QRadioButton("&Cosine");
    radio1->setChecked(true);
    QVBoxLayout *vbox = new QVBoxLayout;
    vbox->addWidget(radio1);
    vbox->addWidget(radio2);
    vbox->addWidget(radio3);
    vbox->addWidget(radio4);
    vbox->addWidget(radio5);
    kernelRadioBox->setMinimumSize(260,140);
    kernelRadioBox->setMaximumSize(260,140);
    kernelRadioBox->setSizePolicy(QSizePolicy::Fixed, QSizePolicy::Fixed);
    kernelRadioBox->setLayout(vbox);

    QButtonGroup *kernelGroup = new QButtonGroup;
    kernelGroup->addButton(radio1, 0);
    kernelGroup->addButton(radio2, 1);
    kernelGroup->addButton(radio3, 2);
    kernelGroup->addButton(radio4, 3);
    kernelGroup->addButton(radio5, 4);
    QObject::connect(kernelGroup, SIGNAL(buttonClicked(int)), this, SLOT(getKernel(int)));

    m_svg = new QSvgWidget();
    runRandomDataCmd();         // also calls plot()

    QGroupBox *estimationBox = new QGroupBox("Density estimation bandwidth (scaled by 100)");
    QHBoxLayout *spinners = new QHBoxLayout;
    spinners->addWidget(spinBox);
    spinners->addWidget(slider);
    QVBoxLayout *topright = new QVBoxLayout;
    topright->addLayout(spinners);
    topright->addWidget(cmdLabel);
    topright->addWidget(cmdEntry);
    estimationBox->setMinimumSize(360,140);
    estimationBox->setMaximumSize(360,140);
    estimationBox->setSizePolicy(QSizePolicy::Fixed, QSizePolicy::Fixed);
    estimationBox->setLayout(topright);
    QHBoxLayout *upperlayout = new QHBoxLayout;
    upperlayout->addWidget(kernelRadioBox);
    upperlayout->addWidget(estimationBox);

    QHBoxLayout *lowerlayout = new QHBoxLayout;
    lowerlayout->addWidget(m_svg);

    QVBoxLayout *outer = new QVBoxLayout;
    outer->addLayout(upperlayout);
    outer->addLayout(lowerlayout);
    window->setLayout(outer);
    window->show();
}

void QtDensity::plot(void) {
    const char *kernelstrings[] = { "gaussian", "epanechnikov", "rectangular", "triangular", "cosine" };
    m_R["bw"] = m_bw;
    m_R["kernel"] = kernelstrings[m_kernel]; // that passes the string to R
    std::string cmd0 = "svg(width=6,height=6,pointsize=10,filename=tfile); ";
    std::string cmd1 = "plot(density(y, bw=bw/100, kernel=kernel), xlim=range(y)+c(-2,2), main=\"Kernel: ";
    std::string cmd2 = "\"); points(y, rep(0, length(y)), pch=16, col=rgb(0,0,0,1/4));  dev.off()";
    std::string cmd = cmd0 + cmd1 + kernelstrings[m_kernel] + cmd2; // stick the selected kernel in the middle
    m_R.parseEvalQ(cmd);
    filterFile();           // we need to simplify the svg file for display by Qt
    m_svg->load(m_svgfile);
}

void QtDensity::getBandwidth(int bw) {
    if (bw != m_bw) {
        m_bw = bw;
        plot();
    }
}

void QtDensity::getKernel(int kernel) {
    if (kernel != m_kernel) {
        m_kernel = kernel;
        plot();
    }
}

void QtDensity::getRandomDataCmd(QString txt) {
    m_cmd = txt;
}

void QtDensity::runRandomDataCmd(void) {
    std::string cmd = "y2 <- " + m_cmd.toStdString() + "; y <- y2";
    m_R.parseEvalQNT(cmd);
    plot();                     // after each random draw, update plot with estimate
}

void QtDensity::filterFile() {
    // cairoDevice creates richer SVG than Qt can display
    // but per Michaele Lawrence, a simple trick is to s/symbol/g/ which we do here
    QFile infile(m_tempfile);
    infile.open(QFile::ReadOnly);
    QFile outfile(m_svgfile);
    outfile.open(QFile::WriteOnly | QFile::Truncate);
    
    QTextStream in(&infile);
    QTextStream out(&outfile);
    QRegExp rx1("<symbol");
    QRegExp rx2("</symbol");    
    while (!in.atEnd()) {
        QString line = in.readLine();
        line.replace(rx1, "<g"); // so '<symbol' becomes '<g ...'
        line.replace(rx2, "</g");// and '</symbol becomes '</g'
        out << line << "\n";
    }
    infile.close();
    outfile.close();
}
#pragma GCC diagnostic pop
```
