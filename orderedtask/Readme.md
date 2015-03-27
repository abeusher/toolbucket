#### orderedtask.Pool

The primary use-case for this package would be to insert tasks in order, have them
processed in parallel and have the results consumed in the original order.  

If a tasks is going to take 10ms and you have 10,000 tasks to perform, then the expected runtime
to perform those tasks one at a time is: `10000 * 10ms == 100 seconds`.  But if you can do them 
parallel with 12 workers, the expect runtime becomes `(10000 * 10ms) / 12 == 8.3 seconds`.

Notes: 

All Tasks are emitted in ascending index order.   i.e. 1, 2, 3,...,N

Internally were using two MinHeaps, one for the lowest task inserted into the pool and 
one for the lowest index of finished tasks.  While the finish tasks index matches the
lowest inserted task, we'll emit the results from the finished tasks.  If the lowest
index for finished tasks is not equal to the lowest inserted task's index, then we wait for
the workers who are processing the lowest indexes to finish before draining.

###### diagram 

```
	        |                            |
	        |     ---->  worker() --\    |
	        |    /                   \   |
Enqueue()--in----------->  worker() ------out----> Results()
	        |    \                   /   |
	        |     ----> worker() ---/    |
	        |                            |
```



#### An example usecase:

You need to read billions of JSON messages out of a Kafka partition, unmarshall the JSON
into a struct and maintain the order of messages.

JSON unmarshalling is very slow at this scale and could add days to your processing time, so you want
to unmarshall the data in parallel.  But sometimes you need those unmarshall messages, in the
original order they arrived in.

Solution use orderedtask.Pool, the Task's index will be the kafka offset and the Task's input
will be the message bytes.   The Task's result will be the unmarshalled struct.  For your
processor func(),  write code to do the JSON unmarshalling into a struct.



###### Example Code:


```
	type Visitor struct {
		Name      string
		ClickLink string
		VisitTime int64
	}
	type Msg struct {
		Offset uint64
		Body   []bytes
	}

	// import github.com/lytics/toolbucket/orderedtask

	const PoolSize = 16

	//Create the pool with PoolSize workers
	pool := orderedtask.NewPool(PoolSize, func(workerlocal map[string]interface{}, t *orderedtask.Task) {
		//For some serialization frameworks you get better performance if you can reuse buffers in a thread-safe manor.
		// workerlocal is used for storing state for each go routine worker.
		/*
			var buf bytes.Buffer
			if b, ok := workerlocal["buf"]; !ok { 
				// This only creates the buffer once, on the first call to process!

				//worker local is not shared between go routines, so it's a good place to 
				// place a store a reusable items like buffers
				buf = bytes.Buffer{}
				workerlocal["buf"] = buf
			} else {
				buf = b.(bytes.Buffer)
			}
			buf.Reset()
		*/

		var visit Visitor
		msg := t.Input.(*Msg)
		err := json.Unmarshal(msg.Body, &vistor)
		if err != nil {
			fmt.Println("error:", err)
		}

		t.Output = visit
	})
	defer pool.Close()

	wg := &sync.WaitGroup{}

	//Produce messages from kafka, into the pool to be unmarshalled
	// and then Consume messages from the pool as structs
	wg.Add(1)
	go func() {
		defer wg.Done()

		kafkaconsumer := createKafkaConsumer() //for example this could be a https://github.com/Shopify/sarama consumer, reading messages from kafka8.

		for {
			select {
			case event := <-kafkaconsumer.Events():
				if event.Err != nil {
					log.Printf("error: consumer: %v", event.Err)
					continue
				}
				//Take the events in from kafka and pass them off to the pool to be unmarshal'ed
				pool.Enqueue(&orderedtask.Task{Index: event.Offset, Input: &Msg{event.Offset, event.Message}})
			case res := <-pool.Results():
				vis := t.Output.(*Visitor)
				fmt.Printf("visitor: off:%v text:%v \n", vis.Name, vis.ClickLink, vis.VisitTime)
				//process the unmarshalled Visitor struct
			}
		}
	}()
```





