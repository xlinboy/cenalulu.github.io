WordCountMapper
```

/**
 * map阶段的业务逻辑写在自定义的map方法中
 * maptask会对每一行输入数据调用一次我们自定义的map()方法
 * @author 13194
 *
 *map和reduce 会遇到网络传输问题
 *
 *KEYIN： 默认情况下，是mr框架所读到的一行文本的起始偏移量，Long,
 *	但是在hadoop中有自己的更精简的序列化接口，所以不直接使用Long， 而用LongWritable
 *VALUEIN：默认情况下，mr框架所读到的一行文件内容，String, 同上，Text
 *KEYOUT：是用户自定义逻辑处理完成后输出数据 key, 在此处是单词， string， 同上Text
 *VALUEOUT：用户自定义逻辑处理完成输出数据中的value, 在此处是单词数， Integer, IntWritable
 */


public class WordCountMapper extends Mapper<LongWritable, Text, Text, IntWritable>{
	
	@Override
	protected void map(LongWritable key, Text value, Mapper<LongWritable, Text, Text, IntWritable>.Context context)
			throws IOException, InterruptedException {
		
		// 拿到第一行数据，转换为string
		String line = value.toString();
		// 根据空格切分
		String[] words = line.split(" ");
		//将单词输出为<单词, 1>
		for (String word : words) {
			// 将单词做为key, 将次数1做为value， 以便后续的数据分发，以便相同的单词到相同的reduce task
			context.write(new Text(word), new IntWritable(1));
		}
	}
	
}

```

WordCountReduce
```
package cn.hadoop.mapreduce;

import java.io.IOException;

import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

/**
 * KEYIN, VALUEIN 对应mapper输入的keyout, valueout
 * KEYOUT,VALUEOUT 自定义reduce逻辑处理输出数据类型
 * keyout, valueout 对应 <单词, 总次数>
 * 
 * @author 13194
 *
 */
public class WordCountReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
	
	/**
	 * 入参key, 是同一组相同单词kv对应的Key
	 */
	@Override
	protected void reduce(Text key, Iterable<IntWritable> values,
			Reducer<Text, IntWritable, Text, IntWritable>.Context context) throws IOException, InterruptedException {
		int count = 0;
		
		for(IntWritable value: values) {
			count += value.get();
		}
		
		context.write(new Text(key), new IntWritable(count));
	}
	
}

```

WordCountDriver
```
package cn.hadoop.mapreduce;

import java.io.IOException;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

/**
 * 相当于一个Yarn集群的客户端
 * 需要在些封装我们的mr程序的相关运行参数，指定jar包
 * 最后提交给yarn
 * @author 13194
 *
 */
public class WordCountDriver {
	
	public static void main(String[] args) throws Exception {
		
		Configuration conf = new Configuration();
		
		Job job = Job.getInstance(conf);
		
		job.setJarByClass(WordCountDriver.class);
		
		//指定业务job要使用的mapper/reducer业务类
		job.setMapperClass(WordCountMapper.class);
		job.setReducerClass(WordCountReducer.class);
		
		//指定mapper输出的数据的kv类型
		job.setMapOutputKeyClass(Text.class);
		job.setMapOutputValueClass(IntWritable.class);
		
		//指定reduce输出的数据的kv类型
		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(IntWritable.class);
		
		
		FileInputFormat.setInputPaths(job, new Path(args[0]));
		FileOutputFormat.setOutputPath(job, new Path(args[1]));
		
		boolean res = job.waitForCompletion(true);
		
		System.exit(res?0:1);
	
	}
}

```



