##Use Case:
	Threshold: 50
	Currently active/activated: 49
	Two listings sent through via REAXML. There will be a race condition where:
	During the product_options resolutions, both requests will get a count of 49 and therefore successfully save both listing as active.
 
##Expected Behaviour:
Based on the above use case, only one listing is made active, the other one will be made in to draft mode.

##Possible Solution:

###Cause of race condition:
When uploading the xml files, rex-reaxml-mapper will publish the listing and decide it to be a draft or active. When listing publisher(rex-reaxml-mapper/lib/rex/reaxml/listing_mapper.rb:67) is created, the listing excess modifier will modify the price. If the price is more than 0, this listing will be published as a draft, otherwise active. 
		
In current senario, publishing a listing and getting the active listing count can be executed at the same time. When an Agent publishes a listing, it may increase the active listing count. If at the same time another listing is under publishing, the active listing count obtained by the agent will not correct. Then the listing will be published as active instead of a draft. Obviously this will not meet the business requirements. 
		
###Why need a lock?
Adding a lock before the publisher is created will prevent others from getting the active listing count or publishing listings through rea-xml. Therefore the status of the listing will be correct. This will solve the race condition.

###Why adding a lock works?
We have tested the row lock(which is 'mysql select for update') on an agent and the results are as following:
		
####Process 1: 
Select for update executing within a transaction

####Process 2
+ 1. select for update executing in a transaction  	- Blocked
+ 2. select within (or not) a transaction			- Nonblocked
+ 3. update within (or not) a transaction			- Blocked

Currenly there is a lock in Agency#purchase and the new lock will be added before publisher is created. So we also tested whether row lock can be acquired recursively.
Here is the example:
<pre><code>
    def try_lock
        ActiveRecord::Base.transaction do
   	    Legacy::Agency.find("A05784", :lock => true)
	    ActiveRecord::Base.transaction do
	        Legacy::Agency.find("A05784", :lock => true)
		p "Oh yes!"
	    end
       	end
    end
</code></pre>

Result: "Oh yes!" is printed out.		 

###Conclusion: 
+ 1. This mechanism can prove that we can prevent multiple xml importing procedures running at the same time. Also, this will have no effect on other reading operations on the locked Agency.
+ 2. Since the lock can be used recursively, we can add a new lock before publisher is created. This will avoid the dead lock.(Note: if cannot recursively, the second lock only gets the resource after the first one releases while the first lock can release the resource only when the second lock releases the resource.)


##Optional Solution:
Another possible solution is to modify the logic when publishing the listing(cp_domain/models/service/listing_publisher.rb:23). But the price needs to be cheked again here to judge whether this listing is published as active or a draft. This solution will result in more code changes and higher complexity.
