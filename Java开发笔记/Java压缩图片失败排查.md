最近生产环境发现一个jpg格式图片压缩失败，然后看了Java压缩的源码，记录下分析过程。之前的代码是设置压缩率然后调用Java工具ImageIO写就行了，但是最近碰到一个7.5M的jpg图片，压缩后生成的文件大小却为0。
```java    
public static byte[] pressImg(File file, long maxLen, String type) {
        byte[] imgByte = null;
        double scale = 0.9;
        if (file.length() > maxLen) {
            do {
                try {
                    BufferedImage bi = ImageIO.read(new BufferedInputStream(new FileInputStream(file)));
                    int width = (int) (bi.getWidth() * scale);
                    int hight = (int) (bi.getHeight() * scale);
                    Image img = bi.getScaledInstance(width, hight, Image.SCALE_SMOOTH);

                    BufferedImage tag = new BufferedImage(width, hight, BufferedImage.TYPE_INT_RGB);

                    Graphics g = tag.getGraphics();
                    g.setColor(Color.red);
                    g.drawImage(img, 0, 0, null);
                    g.dispose();
                    ByteArrayOutputStream bOut = new ByteArrayOutputStream();
                    ImageIO.write(bi, type.toLowerCase(), bOut);
                    bOut.flush();
                    imgByte = bOut.toByteArray();
                } catch (Exception e) {
                   System.out.printf(e.getMessage());
                }
            } while (imgByte.length > maxLen);
        }
        return imgByte;
    }
```
排查发现如果使用`ImageIO.read`获取到的BufferedImage对象就无法写数据到`ByteArrayOutputStream`了，如果使用`new BufferedImage`出来的就不会，对这个问题比较好奇就花了点时间看了下源码。  
重点是` ImageIO`的`write `方法中调用`getWriter(im, formatName)`生成ImageWriter的过程。区别就在这里，如果是通过read生成的BufferedImage就会通过`new ImageTypeSpecifier(image);`创建`ImageTypeSpecifier`，而如果是`new BufferedImage`的就会通过`getSpecifier(bufferedImageType)`方法生成`ImageTypeSpecifier`。
```java
    public static
        ImageTypeSpecifier createFromRenderedImage(RenderedImage image) {
        if (image == null) {
            throw new IllegalArgumentException("image == null!");
        }

        if (image instanceof BufferedImage) {
            int bufferedImageType = ((BufferedImage)image).getType();
            if (bufferedImageType != BufferedImage.TYPE_CUSTOM) {
                return getSpecifier(bufferedImageType);
            }
        }

        return new ImageTypeSpecifier(image);
    }
```
现在看下两种方式创建的源码：
+ new出来的  
可以看到`sampleModel`默认由`SinglePixelPackedSampleModel`实现，其像素深度`private int bitSizes[];`默认为8。
```java
public Packed(ColorSpace colorSpace,
                      int redMask,
                      int greenMask,
                      int blueMask,
                      int alphaMask, // 0 if no alpha
                      int transferType,
                      boolean isAlphaPremultiplied) {
            ......
            this.colorModel =
                new DirectColorModel(colorSpace,
                                     bits,
                                     redMask, greenMask, blueMask,
                                     alphaMask, isAlphaPremultiplied,
                                     transferType);
            this.sampleModel = colorModel.createCompatibleSampleModel(1, 1);
        }

    public SampleModel createCompatibleSampleModel(int w, int h) {
        return new SinglePixelPackedSampleModel(transferType, w, h,
                                                maskArray);
    }        
```
+ read出来的  
由`PixelInterleavedSampleModel`实现，像素深度由文件实际情况决定，7.5M的jpg文件为16。
```java
        public Interleaved(ColorSpace colorSpace,
                           int[] bandOffsets,
                           int dataType,
                           boolean hasAlpha,
                           boolean isAlphaPremultiplied) {
            .......
            this.sampleModel =
                new PixelInterleavedSampleModel(dataType,
                                                w, h,
                                                pixelStride,
                                                w*pixelStride,
                                                bandOffsets);
        }

```
接下来创建`ImageWriter`的时候会根据像素过滤，可以看到`JPEGImageWriterSpi`支持的像素深度为8，16是不支持的，所以没法处理像素深度为16的7.5M的jpg文件。
```java
    public boolean canEncodeImage(ImageTypeSpecifier type) {
        SampleModel sampleModel = type.getSampleModel();

        // Find the maximum bit depth across all channels
        int[] sampleSize = sampleModel.getSampleSize();
        int bitDepth = sampleSize[0];
        for (int i = 1; i < sampleSize.length; i++) {
            if (sampleSize[i] > bitDepth) {
                bitDepth = sampleSize[i];
            }
        }

        // 4450894: Ensure bitDepth is between 1 and 8
        if (bitDepth < 1 || bitDepth > 8) {
            return false;
        }

        return true;
    }

```
总结下：Java中自带的`ImageIO`默认处理jpg格式图片时像素深度为8，read生成的处理器是由图片定义的，如果超过8就无法处理，故需要使用new出来的。