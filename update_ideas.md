## Relevant Rails Models + Associations
New model:
```
class FeatureRating
  belongs_to :activity
  belongs_to :user
  belongs_to :trip_feature
  attribute :rating
  attribute :review_text
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
End


class TripFeature
  attribute :id
  attribute :name # eg "Hiking", "Wine Tasting", "Monkeys"
  has_and_belongs_to_many :activites
  has_many :trip_preferences
  has_many :trips, through: :trip_preferences
end

```

## Activity Review Form Updates

New class to encapsulate TripFeatures business logic to be used by the existing Review Controller
```
class TripActivityFeatures
  #hardcoded or configurable business logic limits
  FEATURE_LIMIT = 3
  SPARSENESS_LIMIT = 2
  ACTIVITY_LIMIT = 3

  def initialize(trip)
    @trip = trip
  end

  # return all sparse trip_features by way of a query across activites_trips
  # and activites_trip_features, group_by trip_feature_id having count <= 2
  def sparse_trip_features_to_review; end

  def sparse_trip_features_for_activity(activity)
    # First get all the sparse features for this trip
    trip_sparse_features = sparse_trip_features_to_review

    # next get all features for this activity that are included in the above list
    features = activity.trip_features.where(trip_feature_id: trip_sparse_features.ids)

    if features.length > 3
        #use ranked_preferences_per_trip_location to apply a ranked order to the
        #features and only take top 3. This could be applied automatically via a #rankings table linked to the Location, but it's likely small/fast #enough to do it in memory to start with
    end
  end

  # return all unique tripfeatures across trips for a location sorted by
  # occurrence count. this could be cached to reduce queries, or stored
  # daily/weekly in a table
  def ranked_preferences_per_trip_location; end
end

```

Rough updates to existing controller
```
class TripReviewController
  private

  #assumes we have a trip and activity already
  def initialize_features_to_rate
    @features_to_rate = TripActivityFeatures.new(@trip).sparse_trip_features_for_activity(@activity)

    previous_ratings = FeatureRating.where(:user_id => @current_user.id, :activity_id => :@activity.id, :trip_feature_id =>[@features_to_rate])
    # create empty ratings for features without ratings and wrap in decorators for view
  End

  #called if the request is a post
  def save_feature_ratings
    # pull ratings data from post content
    # update or create FeatureRatings given current_user, activity,
    # trip_feature, rating, and any review_text
  end
end

```

## Final Review Page Updates

