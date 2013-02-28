# After a week struggling with the terrorists

## Mission (I mean, objectives)

Compare the customer's database with [OFAC][1]'s [Specially Designated Nationals (SDNs) List][2] (so called terrorist watch list) to generate a score measuring the likelihood of an individual being a terrorist. The score ranges from 0 to 100: zero means "ok-guy" and 100 means "OMG s/he is among us".

## Requirements

1. Research (and use) [OFAC gem][3].
2. Allow admin users to enqueue a job that downloads current SDN list from OFAC's site.
3. Allow admin users to enqueue a job that calculates OFAC score per customer.
4. Admin users should be able to access previously created Score Lists.

## Solution

### Research

OFAC gem:

- Downloads current SDN list.
- Generates a CSV.
- Creates a table in MySQL called ```ofac_sdn```.
- Imports data with ```LOAD DATA LOCAL INFILE```.

### Update current SDN List

#### Implementation

##### Database structure

```ruby
class CreateOfacSdns < ActiveRecord::Migration
  def change
    create_table :ofac_sdns do |t|
      t.text :name
      t.string :sdn_type
      t.string :program
      t.string :title
      t.string :vessel_call_sign
      t.string :vessel_type
      t.string :vessel_tonnage
      t.string :gross_registered_tonnage
      t.string :vessel_flag
      t.string :vessel_owner
      t.text :remarks
      t.text :address
      t.string :city
      t.string :country
      t.string :address_remarks
      t.string :alternate_identity_type
      t.text :alternate_identity_name
      t.string :alternate_identity_remarks

      t.timestamps
    end

    add_index :ofac_sdns, :sdn_type
  end
end
```

##### Job

```ruby
class UpdateCurrentSdnListJob < Job
  @queue = :low

  def perform
    OfacSdnLoader.load_current_sdn_file
  end
end
```

##### Controller

```ruby
class SdnListsController < ApplicationController

  def new
  end

  def create
    UpdateCurrentSdnListJob.enqueue
    redirect_to new_sdn_list_path
  end

end
```

##### Tests

- Admins should be able to enqueue the job.

```ruby
describe SdnListsController do

  context "for a full access admin" do
    before do
      # login admin
    end

    it "should enqueue the job" do
      # test if the controller triggers the job
      UpdateCurrentSdnListJob.should_receive("enqueue").exactly(1).times
      post :create
      response.should redirect_to(new_sdn_list_path)
    end
  end

end
```

#### Problems found

1. Cancan attempts to authorize resource, but we don't have a model.

   Solution:

   ```ruby
   class SdnListsController < ApplicationController

     skip_load_and_authorize_resource
     skip_authorization_check

     # ...

  end

2. ```LOAD DATA LOCAL``` disabled in MySQL.

   Solution:

     - Edit ```/etc/mysql/my.cnf``` and add ```local-infile=1``` to ```[client]```, ```[mysqld]```, and ```[mysql]``` sections.
     - Bundle install [mysql2 gem directly from GitHub][4]. Current [mysql2 version in RubyGems.org][5] (0.3.11) does not support local-infile.
     - Add ```local-infile: true``` to ```database.yml```'s different environments.

3. OFAC gem's ```OfacSdnLoader.load_current_sdn_file``` does not return a status. Unsolved.

### Calculate scores

#### Implementation

##### Database structure

```ruby
class CreateScores < ActiveRecord::Migration
  def change
    create_table :scores do |t|
      t.integer :score
      t.date :list_date

      t.references :customer

      t.timestamps
    end

    add_index :scores, :list_date
    add_index :scores, [:list_date, :customer_id], unique: true
  end
end
```

**Alternative**: Creating a ```score_lists``` table, holding Score Lists history. Pros and cons?

##### Model

```ruby
class Score < ActiveRecord::Base
  belongs_to :customer

  validates :list_date, presence: true, uniqueness: {scope: :customer_id}
  validates :customer_id, presence: true

  before_save :calculate_score

  private

  def calculate_score
    ofac_result = Ofac.new(
      name: self.customer.name,
      address: self.customer.address,
      city: self.customer.city)

    self.score = ofac_result.score
  end

end
```

##### Controller

- No controller for ```Score```!
- I created a tableless model to handle virtual Score Lists (see [7 Patterns to Refactor Fat ActiveRecord Models][6]).

```ruby
class ScoreListsController < ApplicationController

  skip_load_and_authorize_resource
  skip_authorization_check

  before_filter :load_score_lists, :only => [:index, :show]

  def index
  end

  def create
    CalculateScoresFromSdnListJob.enqueue(Date.current)
    redirect_to admin_score_lists_path
  end

  def show
    @score_list = ScoreList.new(id: params[:id])
    if @score_list.valid?
      # TODO: scope the following line in Score
      @scores = Score.find_all_by_list_date(@score_list.id)
    end
    render 'index'
  end

  private

  def load_score_lists
    @score_lists = ScoreList.all
  end
end
```

##### Tableless model

```ruby
class ScoreList
  include ActiveModel::Validations
  include ActiveModel::Conversion
  extend ActiveModel::Naming

  attr_accessor :id

  validates :id, presence: true, format: /^\d{8}$/
  validate :id_must_exist_in_scores_as_a_list_date

  def initialize(attributes = {})
    @id = attributes[:id]
  end

  def persisted?
    false
  end

  def date
    return Date.parse(@id)
  end

  def self.all
    list_dates = Score.select('distinct(list_date)').map{ |s| s.list_date }
    collection = []
    list_dates.each do |date|
      collection << ScoreList.new(id: date.to_s(:number))
    end
    collection
  end

  protected

  def id_must_exist_in_scores_as_a_list_date
    if Score.find_by_list_date(id).nil?
      errors.add(:id, 'Invalid id')
    end
  end

end
```

##### Job

```ruby
class CalculateScoresFromSdnListJob < Job
  @queue = :low

  def perform(list_date)
    raise 'SDN list is empty' if OfacSdn.count == 0

    # Run in batches to avoid memory overload
    Customer.find_in_batches(batch_size: 1000) do |customers|
      customers.each do |customer|
        Score.create!(list_date: list_date, customer: customer)
      end
    end
  end
end
```

##### Tests

Analogous to SDN Lists tests.

#### Problems found

1. Encapsulate Score creation process in a transaction?

2. Use reusable scopes and relations (see [10 Ruby on Rails Best Practices][7]).

   In Score model:

   ```ruby
   scope :calculated_on, lambda { |date| where('list_date = ?', date) }
   scope :order_by_score, lambda { |direction = nil| order("scores.score #{direction}") unless direction.nil? }
   ```

3. For testing purposes, how can I instance a new ```Score``` manually setting the protected ```score``` attribute?

   **Solution**:

   ```Score.new(list_date: Date.current, customer: customer).tap { |s| s.score = 50 }```

## Lessons learnt

- How to test if a controllers triggers a job.
- A way to bulk import data to MySQL using Rails.
- Rails' find in batches to avoid memory overload.
- Tableless models using ActiveModel.
- Use scopes to construct better code and more reusable queries.
- Use ```tap``` method to avoid ```MassAssignmentError```s in tests.

[1]: http://www.treasury.gov/ofac "Office of Foreign Assets Control"
[2]: http://www.treasury.gov/resource-center/sanctions/SDN-List/Pages/default.aspx "http://www.treasury.gov/resource-center/sanctions/SDN-List/Pages/default.aspx"
[3]: https://github.com/kevintyll/ofac "OFAC ruby gem"
[4]: https://github.com/brianmario/mysql2
[5]: http://rubygems.org/gems/mysql2
[6]: http://blog.codeclimate.com/blog/2012/10/17/7-ways-to-decompose-fat-activerecord-models/#form-objects
[7]: http://www.sitepoint.com/10-ruby-on-rails-best-practices/
