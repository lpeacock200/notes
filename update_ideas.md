(included code is Ruby/Rails-ish pseudocode)

## Relevant Rails Models + Associations
New model:
```
class FeatureRating
  attribute :rating #smallint
  attribute :review_text #text

  belongs_to :activity
  belongs_to :user
  belongs_to :trip_feature
end

#Should be unique to a user, activity, and feature
add_index :feature_ratings, [:user_id, :activity_id, :trip_feature_id], unique: true
```
Existing models
```
class Activity
  attribute :id
  attribute :name
  has_and_belongs_to_many :trip_features
  has_and_belongs_to_many :trips
  belongs_to :location
end

class Location
  #all trips taking place in this location
  has_many :trips

  #all available activites
  has_many :activites
end

class Trip
  attribute :id
  belongs_to :traveler, class_name: 'User'
  belongs_to :location

  # The list of activities that were included in this traveler's trip
  has_and_belongs_to_many :activities
  has_many :trip_preferences
  has_many :trip_features, through: :trip_preferences
  ...
End

class User
  attribute :id
  attribute :name

  has_many :trips
  # a list of TripFeature that the user is interested in having included in their trip (entered during onboarding)
  has_many :requested_trip_features, class_name: 'TripFeature'
  has_many :feature_ratings
End


class TripFeature
  attribute :id
  attribute :name # eg "Hiking", "Wine Tasting", "Monkeys"
  has_and_belongs_to_many :activites
  has_many :trip_preferences
  has_many :trips, through: :trip_preferences
end

class ActivityBooking
  belongs_to :activity
  belongs_to :user
  belongs_to :trip
  attribute :booking_date
  ...
end

```

## Activity Review Form Updates
Encapsulate the business logic specific to choosing TripFeatures and FeatureRatings for a Trip and/or Activity in a class that can be used by the existing reviews controller.
```
class TripActivityFeatures
  #hardcoded or configurable business logic limits
  FEATURE_LIMIT = 3
  SPARSENESS_LIMIT = 2
  ACTIVITY_LIMIT = 5

  def initialize(trip); end

  # Return sparse features across the given trip. This is done by querying across
  # join tables activites_trips and activites_trip_features, and group_by
  # trip_feature_id having count <= SPARSENESS_LIMIT.
  def sparse_trip_features; end

  # Return all distinct trip_features across trips for a location sorted by
  # occurrence count, as a measure of popularity. This could be cached to reduce
  # queries, or stored daily/weekly in a table.
  def ranked_preferences_per_trip_location; end

  #Return sparse TripFeatures for a given activity within the trip. This involves
  #querying for all the given Activity's TripFeatures that are included in the set of
  #ids returned by sparse_trip_feature. This could be condensed into a single query
  #using a subquery or similar.
  def sparse_trip_features_for_activity(activity); end
end

```

Rough updates to existing controller
```
class TripReviewController
  private

  #called for get request, assumes we have a trip and activity already
  def initialize_features_to_rate
    @features_to_rate = TripActivityFeatures.new(@trip).sparse_trip_features_for_activity(@activity)

    previous_ratings = FeatureRating.where(:user_id => @current_user.id, :activity_id => :@activity.id, :trip_feature_id =>[@features_to_rate])
    # create empty ratings for features without ratings and wrap in decorators for view
  end

  #called if the request is a post
  def save_feature_ratings
    # pull ratings data from post content
    # update or create FeatureRatings given current_user, activity,
    # trip_feature, rating, and any review_text
  end
end

```
Questions:
- Is there a situation where it makes sense to show an existing rating, like if a user returns to the same review url? Assuming yes for now.
- What if a user doesn't leave a rating? Should that score as a FeatureRating with a different rating value to indicate 'skipped'?

## Final Review Page Updates
Update TripActivityFeatures busines logic
```
class TripActivityFeatures
  ...

  # Reuse query from 'sparse_trip_features' with the only change
  # being a return of features with count >= 3. Limit the number returned by
  # ranking against the most popular location features (as in sparse_trip_features_for_activity)
  # and returning just the top ranking ones
  def non_sparse_trip_features(limit_count = FEATURE_LIMIT); end

  #return a hash of features => [activites]
  def activites_by_non_sparse_features
    trip_features = non_sparse_trip_features

    feature_activites = {}
    # for each trip_feature found, get the ranked activites. need to do this query
    # per feature in a loop because doing it across all features in a single query
    # will make separating actives per features difficult/upredictable
    trip_features.each do |trip_feature|
      # Similar to ranking popular features in ranked_preferences_per_trip_location,
      # the strategy here is rank activites per location across all user trips, or
      # all user ActivityBookings, but sort in ascending rank by occurrence count so
      # that we prioritize less popular activites to gain more info. This ranking
      # data can be a cached from a query or stored as a seperate table that only
      # updates daily/weekly/etc. We use this ordered data to sort and limit the
      # activities shown per feature.
        activites = Activity.where(:trip_feature_id = trip_feature.id, :trip_id => @trip.id) # ... also join and order via ranking data
        feature_activites[trip_feature] = activites
    end
    feature_activites
  end
end
```
Updates to new controller for Additional Info section
```
class TripFinalReviewController
  ...

  private

  #used in get request
  def initialize_feature_activites
    trip_features = TripActivityFeatures.new(@trip)

    #get the non-sparse features with applied limit
    @feature_activites = trip_features.activites_by_non_sparse_features

    #below is just prepping data for the view
    #wrap everything in view objects with the found features, activites, and ratings
    @activites_per_feature = {}
    @feature_activites.each_pair do |feature, activities|
        #for each activity/feature/reating, wrap it in a view object
        activity_ratings = []

        #find any existing ratings for this feature, user, and array of activites
        ratings = FeatureRating.where(:user_id => @current_user.id, :activity_id => [activities.ids], :trip_feature_id => feature.id)

        activites.each do |activity|
          #use the found rating or create a new empty one
          actvity_ratings << ActivityRating.new(activity, rating) #view object with rating and trip_feature info
        end
        @activites_per_feature[feature.id] = activity_ratings
    end
  end

  #for post
  def save_feature_ratings
    #receive back a list of ratings data to store as FeatureRating objects
  end
end
```
Questions/thoughts
- Can there ever be existing ratings for these features? If the user could rate from somewhere else, we'd want to filter those selections out of this page.
- From the mockup, we'd need a separate Activity model that represents a specific booking of that Activity by the user on a particular trip, which would include the date and other info. Call it ActivityBooking. We can include that object in the appropriate query result and pass to the view. It shouldn't affect the overall flow of ratings / features for this execise. It's possible it adds one level of indirection as a Trip could have Activites :through => ActivityBookings, etc.
- This ^ assumes we don't handle different FeatureRatings for the same Activity in a different trip for the same user.
- Options for how to limit the activites per feature on the Final Review Page(s):
  - Prioritize activites linked to fewer trips or ActivityBookings. Advantage is allowing new or lesser known activites to get visibility and collecting more data.
  - Prioritize activites with the most trips or ActivityBookings. Advantage is focusing on what most people already like. This may result in a more stagnant user experience as the same activites always rise to the top of searches / automated itineraries.
